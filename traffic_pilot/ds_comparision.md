# DeepStream data plane — comparison: `cpp/ds` (ours) vs the `cpp/deepstream` architecture (theirs)

Unbiased engineering comparison of our DeepStream daemon (`cpp/ds/traffic_pilotd_ds.cpp`, built
and validated on this Orin NX 16GB) against the alternate DeepStream architecture described as
`cpp/deepstream/traffic_pilotd_deepstream.cpp`. Goal: decide what to keep, what to borrow, and
what matters for **benchmarking many camera streams**.

> TL;DR — They are siblings, not rivals: same GStreamer backbone, same zero-copy NVMM path,
> same Redis `traffic:analytics` contract, same FP16, and **both share the same #1 weakness
> (static batch-1 vehicle engine).** Ours is the *validated, lean, low-latency* base that
> actually runs on this box. Theirs is *scaling-engineered* — it has the three levers our base
> is missing (top-K plate gate, PGIE frame-skip, GPU live-view branch) plus a measured scaling
> study. **For multi-camera benchmarking, port their scaling features onto our proven base.**

---

## 1. What is identical (don't re-litigate these)

| Property | Both pipelines |
|---|---|
| Backbone | One GStreamer process, `nvurisrcbin ×N → nvstreammux → PGIE → nvtracker → SGIE-1 → SGIE-2 → probe` |
| Decode | NVDEC (`nvv4l2decoder`), NV12 in NVMM — **zero-copy** through inference |
| Batching (streams) | `nvstreammux` batches N cameras into one buffer, scaled to MUX_WxH on VIC |
| OCR | `cct.onnx` CTC, batch-16; track-keyed `OcrStabilizer` (vote + confirm-and-hold) |
| Plate | `plate_yolo26n_224_dyn.onnx`, dynamic→batch-4/16 |
| Output | Redis `traffic:analytics` (byte-identical schema) + optional Azure Event Hub |
| Precision | **FP16 only** (neither is INT8 yet) |
| **#1 weakness** | **Vehicle PGIE is a static batch-1 engine → nvinfer runs it serially per frame.** Shared by both. |

---

## 2. Where they differ (the whole story)

| Dimension | **Ours — `cpp/ds`** | **Theirs — `cpp/deepstream`** | Who's ahead |
|---|---|---|---|
| Vehicle detector | `yolo26**n**` (nano) | `yolo26**s**` (small) | **Split** — nano = faster/more cams; s = more accurate |
| Tracker | **NvSORT** (lightweight, scales wide) | **NvDCF** (heavier, better re-id) | Split — NvSORT for many cams, NvDCF for quality |
| **Top-K plate gate** | ❌ none — OCRs every vehicle | ✅ `PLATE_TOP_K` (default 4) per frame | **Theirs** (big for multi-cam) |
| **PGIE frame-skip** | `interval=0`; drop at **decoder** (`drop-frame-interval`) | `interval=N` + tracker interpolation (lever) | **Theirs** has both options |
| **Live view** | NvBufSurfTransform → **OpenCV CPU draw → CPU `imencode`**, on the probe thread | on-demand `tee → nvdsosd → **nvjpegenc** (GPU)` branch | **Theirs** (keeps CPU/probe free) |
| Scaling study | informal (`HANDOFF_ORIN.md` notes) | **measured** `SCALING-TO-80-CAMERAS.md` (1/8/15 cam numbers) | **Theirs** |
| Engine mgmt | hand-built `_fp16.engine` | auto per-batch `_bN_gpu0_fp16.engine` | Theirs (cosmetic) |
| **Validated on THIS Orin** | ✅ debugged end-to-end (cct swap, conf-gate bug, live-view crash) | reported numbers, box unknown | **Ours** (real, not paper) |
| OCR confidence gate | ✅ uses **plate-detector conf** (we found+fixed the bug) | same stabilizer — correctness unverified | **Ours** (known-correct) |

### Notes that matter

- **They are not faster because of the vehicle model** — both serialize batch-1. Their throughput
  edge at scale comes from *doing less work* (top-K gate, frame-skip), not from better batching.
- **nano vs s is a real tradeoff, not a loss.** Our nano detector is cheaper per frame, so for a
  pure "how many cameras can one box hold" benchmark, nano is the *more scalable* baseline. Their
  `s` model trades cameras for detection accuracy. Pick per deployment.
- **Our live-view is the weaker design at scale.** It does a GPU→CPU surface transform, OpenCV
  draw, and CPU JPEG encode **on the streaming thread** every live frame. Fine for 1–2 cameras
  (what we validated), but at 8+ cameras with operators watching it back-pressures the pipeline.
  Their `nvdsosd → nvjpegenc` branch keeps OSD + encode on the GPU, off the hot path. **This is
  the cleanest single thing to adopt.**

---

## 3. What's genuinely better in OURS (keep these)

1. **It runs and is validated on the real Orin NX 16GB** (JetPack 6 / DS 7.1 / TRT 10.3). We hit
   and fixed three device-specific blockers theirs' doc doesn't mention:
   - `lprnet_india.onnx` does **not** parse on TRT 10.3 (channel-striding `MaxPool3d`) → we use `cct.onnx`.
   - OCR stabilizer was gating on the meaningless cct softmax (~0.04) → fixed to use the **plate-detector confidence** (matches the OpenVINO daemon). Without this, **zero plates ever confirm.**
   - Direct CPU-map of the block-linear NVMM surface **segfaults** → fixed with `NvBufSurfTransform` to an RGBA surface array.
2. **Lean by default** — nano + NvSORT is the lower-GPU-cost configuration, i.e. the better
   *starting point* for max-camera scaling.
3. **OpenCV live-view is more flexible** for rich overlays (plate text, ROI polygons, per-zone
   colors) than `nvdsosd`'s display-meta — at a CPU cost. Keep it as the "rich annotate" mode,
   make the GPU branch the "scale" mode.

## 4. What's genuinely better in THEIRS (borrow these)

Ranked by value **for multi-camera benchmarking** (your stated goal):

1. **Top-K plate gate (`PLATE_TOP_K`).** ⭐ The single most important borrow. It bounds the
   plate+OCR cost to K ROIs/frame regardless of vehicle density. Our `1_1.mp4` has 10–20 vehicles
   per frame — today we run plate-det + OCR on *all* of them, so cost scales with traffic *and*
   cameras. With top-K it scales with cameras only. Essential before any honest N-camera number.
2. **PGIE `interval=N` frame-skip + tracker interpolation.** ⭐ Documented ×3–4 camera multiplier,
   one config line. We only drop at the decoder (cuts decode+infer but loses the interpolation
   smoothness). Having *both* levers lets us tune for the NVDEC-bound vs GPU-bound regime.
3. **On-demand GPU live-view branch (`tee → nvdsosd → nvjpegenc`).** ⭐ Moves OSD + JPEG off the
   CPU and off the probe thread — removes our biggest multi-camera back-pressure risk.
4. **The scaling methodology itself** (`SCALING-TO-80-CAMERAS`): measure GR3D%, NVDEC session
   ceiling, and per-camera fps as you add streams; know the two hard walls (GPU at ~8 cams,
   NVDEC buffers at ~15). This is exactly the benchmark harness you need.
5. **Dynamic-batch vehicle engine** (re-export `vehicle_yolo26*` with a dynamic batch axis).
   Removes the shared batch-1 serial penalty, ~×1.5–2. **Both pipelines need this.**

---

## 5. The hard ceilings (same for both, on one Orin NX 16GB)

1. **GPU inference** — vehicle model is batch-1 serial; ~8 cams ≈ 92% GR3D in their measurements.
2. **NVDEC decode** — Orin NX has **one** NVDEC (~16× 1080p30 H.264, fewer for H.265). Hard wall
   on raw stream count, independent of inference. They saw stalls (decode buffers) at ~15 cams.

Realistic per-box comfort zone ≈ **8 cameras at full cascade**; ≈ **25–40 cameras** with the
levers stacked (interval skip + top-K + dynamic-batch + INT8 + NVDLA offload). **80 cameras on one
Orin NX is not feasible** — shard across 3–4× Orin NX (or 1–2× Orin AGX 64GB) behind the same
shared Redis/UI, since every daemon emits the identical `traffic:analytics` contract.

---

## 6. Recommendation — the merge, and the benchmark plan

**Don't replace ours with theirs; merge theirs' scaling layer onto our validated base.** Concrete
order of work, each independently benchmarkable:

| # | Change | Effort | Expected gain | Risk |
|---|---|---|---|---|
| 1 | Add **top-K plate gate** in `analytics_probe` (demote all but top-K vehicles by conf before SGIE) | S | Bounds plate/OCR cost; flattens the traffic-density curve | low |
| 2 | Add **GPU live-view branch** (`tee → nvdsosd → nvjpegenc → appsink`), keep OpenCV path as "rich" mode | M | Frees CPU + probe thread under load | med |
| 3 | Set **PGIE `interval`** (env-tunable) + verify NvSORT interpolation | S | ×3–4 cameras | low |
| 4 | **Dynamic-batch** vehicle engine (re-export + trtexec profile) | M | ×1.5–2, removes batch-1 serial | med |
| 5 | **INT8** calibration for vehicle/plate | L | ~×2 | med |
| 6 | Offload a detector to **NVDLA** | L | a whole extra accelerator | high |

**Benchmark harness for multiple cameras (do this first, it needs no code change):** fan `1_1.mp4`
out as N identical sources (duplicate the cam entry in `cameras.json`, or publish N RTSP streams),
then sweep N = 1, 2, 4, 8, 12 while logging `tegrastats` (GR3D%, NVDEC), the daemon's
`vehicle_fps_agg`, and per-camera analytics rate. That gives you our base curve and the two
ceilings — then re-run after each change above to attribute the gain.

> Bottom line: **neither is "completely better."** Ours is the proven, lean, correct-on-this-box
> baseline; theirs is the scaling-instrumented evolution of the same design. The win is our base +
> their top-K gate + GPU live-view + interval lever + the dynamic-batch fix both still need.
