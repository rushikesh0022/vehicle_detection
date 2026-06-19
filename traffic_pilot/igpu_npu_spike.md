# Why iGPU (GR3D) and NVDEC utilisation shift on the Orin

This explains why the **iGPU compute (GR3D)** and **NVDEC decode** percentages on the Bench page
fluctuate — sometimes a lot — even though "we're decoding and inferring every frame, so it should
be constant." Short version: **every frame is processed, but not every frame costs the same**, and
the numbers are sampled over very short windows. None of the wobble below is a bug.

Context: NVIDIA Jetson Orin NX, DeepStream cascade `NVDEC decode → vehicle detect → tracker →
plate detect → OCR`. Metrics come from `tegrastats` (`GR3D_FREQ %` = GPU compute busy,
`NVDEC %` = hardware decoder busy), sampled every 500 ms; the Bench page reads `run/tegrastats.log`.

---

## TL;DR

- **GR3D (iGPU compute) swings with scene content** — plate-detect + OCR run *per vehicle*, so a
  frame with 8 cars costs far more than one with 2. Busy footage → GR3D near 100%; sparse → lower.
- **NVDEC (decode) swings with frame type** — H.264/H.265 keyframes (I-frames) cost several times
  more to decode than P/B-frames, and they recur every 1–2 s → periodic NVDEC spikes.
- **NVDEC and the GPU spike *together*** — NVDEC can only decode into a small fixed pool of frame
  buffers, and the GPU frees those buffers only as it finishes each frame. So when the GPU stalls on
  a heavy frame, decode stalls too; when it catches up, decode bursts. They're separate chips but
  **flow-coupled** — this is why the decode rate isn't flat (see §2).
- **The displayed number is a near-instantaneous sample**, so it looks jumpier than the real average.
- **Sustained throughput drops when the GPU saturates or gets hot** (~86 °C) — that's a real shift,
  not measurement noise, and it's the device hitting its ceiling.

---

## What the two numbers mean

| Metric | tegrastats field | What it is |
|---|---|---|
| iGPU compute | `GR3D_FREQ NN%` | % of the sample the Ampere GPU's graphics/compute engine was busy (all TensorRT inference: vehicle + plate + OCR, + tracker GPU ops). |
| NVDEC decode | `NVDEC NN%` | % of the sample the dedicated hardware video decoder was busy turning H.264/H.265 into frames. |

They are **physically separate engines**: NVDEC only decodes; GR3D only runs the neural nets. A
frame uses NVDEC first, then GR3D. They fluctuate partly for *different* reasons (below) — but they
are **not fully independent**: NVDEC is *flow-coupled* to the GPU through a small shared pool of
frame buffers (see §2), which is why their spikes so often rise and fall together.

---

## Why they fluctuate (the real reasons)

### 1. Per-frame work is not constant — even though every frame is processed
- **GR3D / inference:** the vehicle detector runs on every frame (roughly constant), **but plate
  detection and OCR run once per detected vehicle.** Traffic density changes shot to shot, so the
  secondary-stage work — and thus GR3D — rises and falls with how many vehicles are on screen.
- **NVDEC / decode:** a compressed video stream is a mix of **I-frames (keyframes)** and
  **P/B-frames**. An I-frame is a full picture and is **much more expensive to decode**; P/B-frames
  are cheap deltas. Keyframes appear periodically (often every 1–2 s), so NVDEC shows a regular
  spike-then-settle pattern. Scene motion/detail also changes per-frame decode cost.

So "processing every frame" ≠ "constant load" — the *amount* of work per frame genuinely varies.

### 2. NVDEC and the GPU are coupled by a shared pool of frame "slots" — why decode spikes instead of staying flat
NVDEC is separate, fast silicon — on its own it could decode *hundreds* of fps and wouldn't care
what's *in* a frame. So why doesn't the decode rate stay flat at the source rate? Because **NVDEC
has to decode each frame *into* a buffer (a "slot"), and there is only a small, fixed pool of them.**
A slot is freed only **after the GPU has finished the whole cascade on that frame** (vehicle + plate
+ OCR) and it leaves the pipeline.

So when the GPU hits a **heavy frame** (many vehicles → slow plate+OCR), it holds its slots longer,
the pool empties, and **NVDEC stalls — not because it's slow, but because it has nowhere to put the
next frame.** Decode dips. When the GPU clears the backlog, several slots free at once and **NVDEC
refills them in a burst** → decode spikes. That stall-then-burst is why the decode rate (and the
NVDEC %) rides up and down **in step with the GPU** instead of sitting flat — even though NVDEC
itself "doesn't care" about vehicles.

> **Parking-garage analogy:** NVDEC is a valet parking cars (frames) into a small garage. A spot
> frees only when the GPU "inspector" finishes with that car. A heavy car → the inspector is slow →
> the garage fills → the valet has to *wait* (nowhere to park). Then a batch clears, spots open, and
> the valet *rushes* to catch up. Over a minute he parks exactly at the arrival rate; second to
> second he looks bursty.

This coupling is **deliberate**: the buffer between decode and inference is kept **shallow**
(`q_pgie max-size-buffers=4` in `cpp/ds/src/traffic_pilotd_ds.cpp`) so the system stays
**low-latency** — it never works on stale, seconds-old frames. A huge buffer *would* flatten the
decode line, but every frame would then be seconds old by the time its plate is read. So the two
engines are physically independent but **flow-coupled** through that small pool — the real reason
they appear to spike together all the time.

### 3. Frames arrive in bursts, not perfectly evenly
A live RTSP source is buffered (jitter-buffer, ~500 ms) and delivered in small bursts; the muxer
batches across cameras (`batched-push-timeout`). The GPU therefore works in **bursts with short
gaps** rather than a perfectly flat line. A 500 ms tegrastats sample lands on different phases of
that burst → different %.

### 4. The displayed value is a near-instantaneous sample
`tegrastats` reports the busy-fraction of each **500 ms** sample, and the Bench page's live poll
shows essentially **one sample per update**. So it jumps (e.g. NVDEC 45% then 84%) where the true
multi-second **average** is steady (~65%). This is the single biggest reason it *looks* erratic.
Averaging over a few seconds removes most of the apparent jitter.

### 5. Saturation + thermal — a *real* downward shift (not noise)
When GR3D sits at **85–98%**, the GPU is essentially maxed. Push more work (busier footage, more
cameras) and it **can't keep up → frames are dropped** (`decode/inferred` fall below `offered`,
`lost > 0`). Separately, after sustained load the chip heats toward **~86–87 °C (its thermal
limit)** and the SoC pulls back to protect itself — so a config that ran 100 fps *cold* settles
lower *hot*. This is the device's genuine ceiling, and it shifts with **scene content and
temperature**, not just measurement.

### 6. "Decode > offered" spikes — yes, it's the *previous* (backlogged) frames being counted now
You'll sometimes see the decode rate read **higher than the frame rate you're actually sending** —
e.g. the camera sends **12 fps** but a window reports 18–24, or a 25 fps camera reads `decfps=26`.
**This is exactly the previous frames catching up — not new frames, and not the same frame counted
twice.**

How it happens: the decode rate is measured as *(frames that crossed the muxer in this window) ÷
(window seconds)*. When the pipeline briefly stalls — the GPU was holding the frame slots (§2), or a
jitter-buffer/burst gap (§3) — a few frames **back up in the buffers** instead of crossing during
that window, so that window reads **low**. The instant the stall clears, the **backlog flushes
through together** and all of those frames get counted in the *next* window. So that window's count
is "its own frames **+** the leftovers from the previous window," and the rate reads **above** the
source fps. Then it dips again. Up, down, up, down — around the true average.

So for a "frame budget" of 12 fps: a window can show ~20 because it's draining the 8-or-so frames
that piled up while the GPU was busy a moment earlier. Those are real frames that were already
decoded/queued — just *counted late*.

Two things to be clear about:
- **No frame is counted twice.** Each frame is counted exactly once; the "overflow" is only about
  *when* it's counted — backlogged frames cluster into one window instead of spreading evenly.
- **It can never exceed the source rate *on average*.** A 12 fps camera averages 12; a 25 fps camera
  averages 25 — you can't decode more frames than the camera sent. Each per-window overflow is
  *borrowed* from a neighbouring (low) window, not created. Once the pipeline runs with no stalls,
  the reading locks to the true rate.

---

## How to read the numbers correctly

- **Trust the multi-second average, not a single 1 s sample.** Decode ~100 with NVDEC ~65% and
  GR3D ~85% is a healthy steady state even if individual samples bounce.
- **`offered` vs `decoded` vs `inferred` vs `lost`** is the honest capacity readout:
  - `decoded ≈ inferred ≈ offered`, `lost ≈ 0` → the device is keeping up.
  - `inferred < offered`, `lost > 0` → the GPU is at its ceiling (saturated and/or thermal).
- **A short jump in GR3D/NVDEC %** = normal per-frame variation. **A sustained fall in `inferred`
  with rising `lost`** = a real limit being hit.

## Reducing the swing / raising the ceiling

- **Smoother readout:** average tegrastats over ~2–3 s instead of one sample (display-only).
- **Less thermal throttling:** ensure the fan runs at max (`sudo jetson_clocks --fan`); good airflow
  keeps `tj` below the throttle point so sustained fps stays near the cold-start number.
- **More headroom:** fewer cameras, lower fps, or `PGIE_INTERVAL=1`/`2` (run the detector every
  2nd/3rd frame, tracker interpolates) — cuts GR3D load and the per-frame swing.

---

## One-line summary

GR3D tracks **how busy the scene is** (vehicles → plate+OCR work); NVDEC tracks **frame type**
(keyframes are expensive); the two **spike together** because NVDEC can only decode into a small pool
of frame buffers that the GPU frees as it finishes work (so decode is flow-coupled to the GPU, not
flat); both are sampled over 500 ms so single readings look jumpy; and a *sustained* drop in
throughput is the GPU hitting its **compute/thermal ceiling**, not a glitch.
