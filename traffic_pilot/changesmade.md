# Changes — file:// playback fix, benchmark tooling, and the OpenVINO-only pipeline

A plain summary of everything changed in this work, what each change does, and how it affects the
rest of the system. Nothing was deleted; every change is additive or backward-compatible.

---

## 1. `file://` real-time pacing + seamless loop (the "video freezes" fix)

**File:** `cpp/traffic_pilotd.cpp` (and mirrored in the new OpenVINO daemon).

**What it did before:** a local MP4 was treated like a live RTSP camera — decoded as fast as the
hardware could go (`sync=FALSE`), and on end-of-stream the pipeline was torn down and reconnected
with a growing `2s → 4s → … → 60s` backoff. A short clip therefore flashed a few frames, then
**froze for up to 60 seconds**, repeatedly.

**What changed:** for `file://` sources only —
- the sink now uses `sync=TRUE`, so the clip plays at its **native framerate**, and
- on EOS the pipeline **seeks back to the start** and keeps playing (seamless loop), instead of
  tearing down.

**Effect:** an MP4 now behaves like a continuous live camera. RTSP cameras are **untouched**
(still `sync=FALSE` + reconnect-with-backoff). Toggle with `FILE_REALTIME=0` / `FILE_LOOP=0`.

---

## 2. Benchmark frontend + collaborative cascading log

**New files:** `services/config_api/bench_capture.py`, `services/config_api/routers/bench.py`,
`tools/bench/cascade_bench.py`, `ui/src/pages/Bench.tsx`.
**Touched:** `services/config_api/main.py` (register router), `ui/src/App.tsx` + `ui/src/components/Layout.tsx`
(route + nav), `ui/src/lib/api.ts` (client + types).

**What it does:** a clean `/bench` page shows the cascade (decode → vehicle → plate → OCR) per
stage, plus a **collaborative log** (`bench/cascade_log.jsonl`) of every run. Each capture
**measures the live daemon's own counters** over a window and is **tagged by backend mode**
(`hybrid` / `all_axelera` / `all_openvino`), so Axelera and OpenVINO runs accumulate in one file
and compare side by side.

**Effect:** no change to analytics/UI plumbing — it reads the same `run/daemon.log` counters +
Redis the daemon already produces. Capture via the page button or
`python tools/bench/cascade_bench.py --window 15`.

---

## 3. N-camera load generator (one clip → many cameras)

**New files:** `tools/bench/make_cameras.py`, `config/cameras-15-plate.json`.

**What it does:** generates an N-camera config where every camera points at the same video file.
Combined with the `file://` fix, each camera is an independent paced+looped pipeline — so one clip
load-tests the whole cascade at fleet scale. Used to make the 15× `number_plate.mp4` config.

**Effect:** lets you benchmark 15 cameras from a single MP4 with no RTSP simulator. Regenerate for
any N: `python tools/bench/make_cameras.py --n 30 --video file:///abs/clip.mp4 --out config/…json`.

---

## 4. The OpenVINO-only data plane (the main piece)

**New files:** `cpp/ov_vehicle.hpp`, `cpp/traffic_pilotd_openvino.cpp`,
`cpp/run_openvino_daemon.sh`, `start_openvino.sh`.
**Touched:** `cpp/CMakeLists.txt` (new target, **no Axelera libs**), `cpp/ov_infer.hpp`
(`add_npu_platform` now honors `NPU_COMPILER_TYPE`).

**What it does:** a second daemon, `traffic_pilotd_openvino`, that runs the **entire cascade on
the Intel side** — vehicle detection on the **iGPU** (new `OVVehicle`, mirrors the existing
`OVPlate`), plate detection + OCR on the **NPU** (reusing `OVPlate`/`OVOcr`). It does **not** link
`libaxruntime`, so:
- the **NPU runs in-process** — no `npu_worker` subprocess, no Unix-socket IPC; and
- the Axelera AIPU is never touched.

It reuses the tracker, stabilizer, publisher, live-view, and `build_and_publish` logic, and emits
**byte-identical Redis messages and stat-log lines** as the mixed daemon.

**Effect:** the UI, analytics consumer, Event Hub path, and `/bench` capture all work with **zero
changes** — only the inference backend differs. Run it with
`CAMERAS_FILE=… ./start_openvino.sh` (vs `./start.sh` for Axelera); stop with the shared
`./stop.sh`. Device per stage is env-tunable without a rebuild
(`VEHICLE_DEVICE`/`PLATE_DEVICE`/`OCR_DEVICE`, `VEHICLE_ONNX`).

### NPU enablement (the three things that were load-bearing)
1. **Clean `LD_LIBRARY_PATH`** = OpenVINO libs + `/usr/lib/x86_64-linux-gnu` only. Any
   `/opt/axelera/runtime` path leaking in breaks NPU enumeration.
2. **`NPU_COMPILER_TYPE=DRIVER`** — this OpenVINO build ships no MLIR compiler-loader; the
   in-driver compiler (`libze_intel_npu.so`) is used instead.
3. **Pre-compile the NPU blob once** before the worker pool starts (so workers import, not race),
   and leak the warm objects (`_exit(0)` skips the crashing NPU teardown).

All three are handled automatically by `cpp/run_openvino_daemon.sh`.

---

## 5. Documentation

**New files:** `docs/cpp/architecture_openvino.md` (HLD, mirrors `01-ARCHITECTURE.md`),
`docs/cpp/deploy_openvino.md` (deploy guide, mirrors `03-DEPLOY.md`), and this file.

---

## Benchmark result (15× `number_plate.mp4`)

| Backend | Vehicle engine | Plate/OCR | Decode fps | Vehicle fps (agg) | fps/cam |
|---------|----------------|-----------|-----------:|------------------:|--------:|
| Axelera hybrid | AIPU | AIPU / NPU | 360 | **180** | 12.0 |
| All-OpenVINO | **iGPU** | **NPU / NPU** | 297 | **116** | 7.8 |

The iGPU vehicle model is the binding stage at ~65% of the AIPU's vehicle throughput; plate/OCR on
the NPU read plates cleanly either way. The full, regenerable comparison lives on `/bench`.

---

## No hardcoding (confirmed)

- **The benchmark numbers are measured, not written down.** Every fps is a counter delta over a
  wall-clock window (`decode_fps = Δ(pushed+skipped)/dt`, `vehicle_fps = Δdelivered/dt`); there is
  no `180`/`116`/`12` literal anywhere. Proof: the same code reported 5.3, 20, 116, and 180 fps in
  different conditions — it varies with the live system.
- **The OpenVINO daemon has no hardcoded results, fps, resolutions, or paths.** Frame size is read
  from the GStreamer caps; the gate rate comes from each camera's config `infer_fps`; model paths
  default but are overridable via argv/env; device per stage is env-driven. (Grep for hardcoded
  fps/resolution/result literals in `traffic_pilotd_openvino.cpp` / `ov_vehicle.hpp` returns
  nothing; no absolute paths in the C++.)
- The **run scripts** contain box-specific paths (e.g. `/home/admin1/voyager-sdk`) exactly like
  the existing `run_daemon.sh` — that's environment configuration, not result hardcoding.

---

## Known limitation — jerky live view at high camera counts (NOT a benchmark issue)

When the OpenVINO-only daemon runs **15 cameras**, the iGPU vehicle stage is saturated and becomes
the bottleneck, so two things make the **live preview** stutter / "jump forward" instead of playing
linearly:

1. The source is paced to ~24 fps but the GPU processes far fewer, so **most frames are dropped**,
   and the drops are bursty (GPU scheduling) — the displayed content skips ahead in time.
2. Cameras are routed to 4 worker threads (`sid % 4`), each handling 3–4 cameras FIFO, so per-camera
   delivery is **uneven** (measured spread ~1700–5000 frames over the same window) — some cameras
   look smooth, others stutter.

This does **not** affect the benchmark — aggregate throughput is still measured correctly; it only
affects how smooth the *preview* looks. For smooth, linear playback, run **fewer cameras** (e.g. 4–6
so the GPU sustains the full 12 fps/cam), or use the lighter `vehicle_yolo26n` model (already the
default). It is a symptom of the iGPU being the binding stage at fleet scale, which is the very
thing the benchmark is measuring.
