# Deploy Guide — OpenVINO-Only

How to build and run `traffic_pilotd_openvino` on a box — the all-Intel data plane (vehicle on
iGPU, plate + OCR on NPU, no Axelera). This is the sibling of [03-DEPLOY.md](03-DEPLOY.md); the
architecture is [architecture_openvino.md](architecture_openvino.md). The non-obvious part is the
**NPU environment** — and it is *different from* (and simpler than) the mixed daemon's: a **clean**
Level-Zero path and the **in-driver compiler**, no 1.21.9 pin. Get that right and the rest is
standard.

## 1. Hardware

| Component | Requirement | Verify |
|-----------|-------------|--------|
| CPU + iGPU | Intel Core Ultra (Arrow/Meteor/Lunar Lake) with Arc iGPU | `lspci | grep -i vga` |
| NPU | Intel AI Boost (NPU) — **strongly recommended** for plate/OCR | `lspci | grep -i "NPU"`, `ls /dev/accel/accel0` |
| AIPU | **Not required.** The Axelera Metis is never used. | — |
| RAM | 16 GB+ (more for many cams) | |

The NPU is optional: if it is absent or fails to compile, `OVPlate`/`OVOcr` **fall back to the
iGPU automatically** (lower throughput, since the iGPU then does vehicle *and* plate *and* OCR —
see §8). For the intended benchmark, you want the NPU.

## 2. System packages & drivers

```bash
# Redis (the analytics bus) + ffmpeg (snapshots / RTSP simulator)
sudo apt install redis-server ffmpeg

# Intel NPU stack — user-mode driver + firmware. Provides /dev/accel/accel0,
# libze_intel_npu.so (the in-driver NPU compiler this daemon uses), and the NPU firmware.
sudo apt install intel-level-zero-npu intel-fw-npu

# The user must be in 'render' to open /dev/accel/accel0 and the iGPU
sudo usermod -aG render "$USER"   # re-login after
```

- **OpenVINO 2026.2** must be available with C++ headers/libs and the **`intel_npu` plugin**
  (`libopenvino_intel_npu_plugin.so`). On this box it lives inside the Voyager venv
  (`venv/lib/python3.12/site-packages/openvino`); CMake finds it via `OpenVINOConfig.cmake`
  there. A standalone OpenVINO runtime install works too — the daemon does **not** need the
  Voyager SDK, only OpenVINO. (We still `source` that venv simply because that is where this
  box's OpenVINO is installed.)
- Unlike the mixed daemon, **no `libze_loader.so.1.21.9` pin is needed.** This daemon uses the
  **OS-default Level-Zero loader** (1.24.2 here) plus the NPU's in-driver compiler. See §4.
- The MLIR compiler-loader (`libopenvino_intel_npu_compiler_loader.so`) is **not required** —
  it is absent from this OpenVINO build, which is exactly why we pin the DRIVER compiler (§4).

## 3. Build

```bash
source /home/admin1/voyager-sdk/venv/bin/activate
cmake -S cpp -B cpp/build -DCMAKE_BUILD_TYPE=Release
cmake --build cpp/build --target traffic_pilotd_openvino
```

The `traffic_pilotd_openvino` target links **only** `gstreamer-1.0` + `-app`/`-video`/
`-allocators`, `opencv4`, `OpenVINO`, `OpenSSL`, `pthread` — **no `axruntime`/`axstreamer`**
(verify: `ldd cpp/build/traffic_pilotd_openvino | grep -i axruntime` → empty). That zero-Axelera
linkage is what lets the NPU run in-process.

> Note: the top-level `cpp/CMakeLists.txt` still `pkg_check_modules(axruntime REQUIRED ...)` for
> the *other* targets, so on a box with **no** Axelera SDK at all the configure step fails before
> reaching this target. To build OpenVINO-only on an Axelera-free box, make that `axruntime`
> check optional (guard the `alpr_poc`/`traffic_pilotd` targets behind it). On this box the SDK
> is present, so configure succeeds as-is.

## 4. The NPU recipe (why `run_openvino_daemon.sh` is the way it is)

Getting the NPU to enumerate **and compile** in this process needs three things — all handled by
`cpp/run_openvino_daemon.sh` + `cpp/ov_infer.hpp`, no manual steps:

1. **Clean `LD_LIBRARY_PATH`.** Set it to **OpenVINO libs + `/usr/lib/x86_64-linux-gnu` only**.
   Do *not* inherit a polluted path: any `/opt/axelera/runtime` entry leaking in makes the NPU
   plugin fail to enumerate (the AIPU runtime's bundled Level-Zero loader shadows it) — you'll
   see only `['CPU','GPU']`. The run script *overwrites* `LD_LIBRARY_PATH`, so it is robust
   regardless of how it's launched.
2. **`NPU_COMPILER_TYPE=DRIVER`.** This OpenVINO build ships no standalone MLIR compiler-loader
   `.so`, so the default/MLIR path errors with *"Cannot load libopenvino_intel_npu_compiler_loader.so"*.
   Forcing the **in-driver** compiler (bundled in `libze_intel_npu.so`) compiles cleanly. The
   value is passed through to the NPU config by `ov_infer.hpp::add_npu_platform`.
3. **Pre-compile the NPU blob once.** Before spawning the `PLATE_WORKERS` pool, the daemon
   compiles plate + OCR on the NPU a single time and exports the blob; the workers then **import**
   it (fast, and no N-way race to write the same blob file). The warm objects are intentionally
   leaked — destroying an NPU-compiled model mid-process corrupts the heap.

Plus: `_exit(0)` at shutdown skips the OV/L0 static teardown (which otherwise corrupts the heap),
and `OV_CACHE_DIR` caches the iGPU compiles across restarts. **No `npu_worker` subprocess** — the
mixed daemon needs it only to escape `libaxruntime`'s loader; this binary has no such conflict.

## 5. Run

Full stack (control plane + UI + the OpenVINO data plane), tags `/bench` runs `all_openvino`:

```bash
CAMERAS_FILE=$PWD/config/cameras-15-plate.json ./start_openvino.sh
#   the Axelera hybrid equivalent is:  CAMERAS_FILE=... ./start.sh
# stop either stack with the shared ./stop.sh
```

Daemon only (control plane already up):

```bash
./cpp/run_openvino_daemon.sh config/cameras.json 600     # run 600 s
```

The AIPU is not used, so there is no device-exclusivity issue — but only **one** data-plane
daemon should publish to `traffic:analytics` at a time, so `./stop.sh` any running hybrid first.
`start_openvino.sh` also writes `run/system.json` = `{"backend_mode":"all_openvino"}` so the UI
and the `/bench` capture label the run correctly.

## 6. Environment levers (override per run)

| Var | Default | Effect |
|-----|---------|--------|
| `VEHICLE_DEVICE` | `GPU` | OpenVINO vehicle-detector device (`GPU`/`NPU`/`CPU`). The binding stage. |
| `PLATE_DEVICE` | `NPU` | OpenVINO plate-detector device. |
| `OCR_DEVICE` | `NPU` | OpenVINO OCR device. |
| `VEHICLE_ONNX` | `…/vehicle_yolo26n_640.onnx` | Vehicle model. Use `vehicle_yolo26s_640.onnx` for accuracy over speed. |
| `VEHICLE_WORKERS` | `4` | iGPU vehicle infer pool (cameras routed `sid % N`). |
| `PLATE_WORKERS` | `4` | NPU plate+OCR worker pool. |
| `TOP_K` | `4` | Plate fan-out per frame (largest in-ROI vehicles). |
| `NPU_COMPILER_TYPE` | `DRIVER` | NPU compiler path (keep `DRIVER` on this build). |
| `NPU_PLATFORM` | `3720` | NPU platform id passed to the compiler. |
| `GATE_TOL` | `0.3` | fps-gate jitter tolerance. |
| `LIVE_FPS` / `LIVE_MAX_W` | `12` / `960` | Annotated live-view rate / max JPEG width (only watched cams pay). |
| `OV_CACHE_DIR` | `run/ov_cache` | OpenVINO compiled-blob cache. |
| `AZURE_EVENT_HUB_CONNECTION_STRING` / `_NAME` | unset | Enables the Event Hub sink (same payload as Redis). |

## 7. Wiring the endpoints

**Cameras.** Each camera's `uri` is anything `uridecodebin` opens — `rtsp://user:pass@host:554/path`
for live cameras, `file:///path` for a clip (paced to real time + looped). Author cameras in
`config/cameras.json` (or via the UI). For an N-cam load test off one clip:
`python tools/bench/make_cameras.py --n 15 --video file:///abs/clip.mp4 --out config/cameras-15-plate.json`.

**Azure Event Hub.** Identical to the mixed daemon — off unless `AZURE_EVENT_HUB_CONNECTION_STRING`
(+ `_NAME` if no `EntityPath`) is set; same `camera_analytics` JSON, batched over HTTPS via
OpenSSL, never blocks Redis.

## 8. End-to-end verification (deployment checklist)

1. **iGPU + NPU present.** `ls /dev/accel/accel0` exists; you're in `render` (`id | grep render`).
2. **NPU enumerates under OpenVINO with a clean path:**
   ```bash
   OVLIBS=$HOME/voyager-sdk/venv/lib/python3.12/site-packages/openvino/libs
   LD_LIBRARY_PATH="$OVLIBS:/usr/lib/x86_64-linux-gnu" \
     python3 -c "from openvino import Core; print(Core().available_devices)"
   ```
   → must list `'NPU'`. If it shows only `CPU`/`GPU`, an Axelera path is leaking into the env (§4).
3. **Build current:** `cmake --build cpp/build --target traffic_pilotd_openvino` → `Built target`,
   and `ldd … | grep -i axruntime` is empty.
4. **Daemon starts on the right engines.** `./stop.sh`, then `./cpp/run_openvino_daemon.sh
   config/cameras.json 60`. Watch stderr for:
   - `OpenVINO devices: CPU GPU NPU` (NPU present),
   - `stages: vehicle=GPU x4 | plate=NPU | ocr=NPU`,
   - `[plate] compiled on NPU` + `[ocr] compiled on NPU` once, then `imported NPU blob on NPU`
     for each worker. A `GPU fallback` line means the NPU isn't really in play (see §4).
5. **Decode + GPU keeping up:** per-camera stats show `delivered` advancing; `OV-callback:` shows
   a sane `ms/frame` (≈ 30–40 ms at 15 cams on the iGPU). A high `ms/frame` with low fps means the
   GPU is saturated — confirm plate/OCR really landed on the NPU (step 4), not the GPU.
6. **Analytics flowing:** `redis-cli XLEN traffic:analytics` grows; `redis-cli XREVRANGE
   traffic:analytics + - COUNT 1` shows your `camera.id`, `objects[]`, and `license_plates`
   (`confidence:1.0`) on cams with readable plates.
7. **UI + bench:** open the UI — Monitor shows per-camera fps, LiveView shows the annotated MJPEG;
   the **Bench** page (`/bench`) shows the run tagged `all_openvino` next to any Axelera runs.

If 1–7 pass, the OpenVINO-only pipeline is working end to end.

## 9. Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| NPU never enumerated (`['CPU','GPU']`) | A `/opt/axelera/runtime` path leaked into `LD_LIBRARY_PATH`. The run script sets it clean; if launching by hand, use `OVLIBS:/usr/lib/x86_64-linux-gnu` only. §4.1 |
| `Cannot load libopenvino_intel_npu_compiler_loader.so` → GPU fallback | MLIR compiler path attempted. Set `NPU_COMPILER_TYPE=DRIVER` (the default in the run script). §4.2 |
| Vehicle `ms/frame` high, fps low | Plate/OCR fell back to GPU, so the iGPU is doing all three stages. Fix the NPU (steps 2/4); or accept it and lower cam count / `VEHICLE_WORKERS`. |
| `pushed` ≫ `delivered` (big gap) | The iGPU vehicle stage is the wall — expected at high cam counts. Use `vehicle_yolo26n_640.onnx`, raise `VEHICLE_WORKERS` (to a point), or lower per-cam `infer_fps`. |
| Stream/Monitor empty in UI | Redis unreachable, or the config-api/UI from `start_openvino.sh` not running. |
| `corrupted size vs prev_size` at exit | OV/NPU static teardown — must exit via `_exit(0)` (already the case). |
| Two daemons' data mixed | A hybrid (`traffic_pilotd`) and this one both publishing. `./stop.sh` before switching. |
