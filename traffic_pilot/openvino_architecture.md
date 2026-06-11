# Architecture — OpenVINO-Only (HLD)

This is the all-Intel sibling of [01-ARCHITECTURE.md](01-ARCHITECTURE.md). Same ALPR cascade
(vehicle → plate → OCR → events), same Redis/UI contracts — but **no Axelera/Voyager**. It is a
separate executable, `traffic_pilotd_openvino`, used to benchmark the pipeline when every
inference stage runs on the Intel side of the box. The plan it implements is
[08-OPENVINO-ONLY-ARCHITECTURE.md](08-OPENVINO-ONLY-ARCHITECTURE.md); this doc describes what
was actually built.

## 1. Goal & the core idea

Run the full ALPR cascade for N cameras using **only the Intel engines** — CPU + iGPU + NPU —
so the Metis AIPU is never touched. The point is a clean head-to-head: how far does the box go
with no dedicated AI accelerator? The box has three engines in play here, and the design keeps
all three busy at once:

| Engine | Hardware | Job in this pipeline |
|--------|----------|----------------------|
| **iGPU** (Intel Arc) | fixed-function video block + EUs | HW-decode the streams **and** run the vehicle YOLO (OpenVINO, the heavy detector) |
| **NPU** (Intel AI Boost) | OpenVINO 2026.2, in-driver compiler | Plate detection + OCR (the two small models) |
| **CPU** (Core Ultra 9) | the C++ daemon threads | Decode orchestration, tracking, stabilisation, JSON, Redis, draw |
| ~~AIPU~~ (Axelera Metis) | — | **Unused.** `libaxruntime` is not linked, by design. |

The mixed daemon puts the heavy vehicle model on the AIPU and leaves the iGPU for decode only.
Here the iGPU does **both** decode and vehicle detection, so the **iGPU is the binding stage**.
Plate-det and OCR move to the otherwise-idle NPU. Critically, because no `libaxruntime` is
resident, the NPU enumerates and runs **in-process** — there is no `npu_worker` subprocess and
no Unix-socket IPC (that machinery only exists to escape the AIPU runtime's Level-Zero loader).

## 2. Component diagram

```
                         ┌──────────────────── one box (Intel only) ───────────────────┐
  N× file/RTSP H264/H265 │                                                              │
  cameras  ──────────────┼─► uridecodebin (iGPU HW decode) ──► NV12                     │
                         │            │                                                 │
                         │            ▼                                                 │
                         │     per-camera fps gate ──► routed to worker (sid % N)        │
                         │            │                                                 │
                         │            ▼                                                 │
                         │     OVVehicle worker pool (OpenVINO, iGPU)  ◄── binding stage │
                         │       map NV12→BGR · detect [1,300,6] · track · top-K         │
                         │                    │                       │                 │
                         │            ┌───────┘                       └───────┐         │
                         │            ▼                                       ▼         │
                         │      plate pool (M):                          publisher      │
                         │      OVPlate  (OpenVINO NPU)                   pool (2)       │
                         │      OVOcr    (OpenVINO NPU)                      │           │
                         │            │  raw reads                          │           │
                         │            ▼                                     ▼           │
                         │      OcrStabilizer ──────────────────► camera_analytics JSON │
                         │     (confirm & hold)                            │            │
                         │   live-view thread ─► live:frame:<cam>          │            │
                         │                                                 ▼            │
                         │                                     ┌──────────────────┐     │
                         │                                     │      Redis       │◄────┼── UI reads
                         │                                     │ traffic:analytics│     │
                         │                                     └──────────────────┘     │
                         │                                                 │            │
                         │                                      Azure Event Hub (opt)   │
                         └───────────────────────────────────────────────────────────────┘
```

## 3. The pipeline, stage by stage

1. **Ingest + decode (iGPU).** One `uridecodebin` per camera auto-plugs the right HW decoder
   (mixed H264/H265). Output is NV12. `file://` sources are paced to real time and looped on
   EOS so one MP4 behaves like a continuous live camera (shared with the mixed daemon).
2. **Per-camera fps gate (CPU).** A schedule-based rate limiter drops frames arriving faster
   than `infer_fps`, so each camera offers ~12 fps. Identical to the mixed daemon.
3. **Route to a vehicle worker (CPU).** Each accepted frame is pushed to one fixed worker by
   `sid % VEHICLE_WORKERS`, so a camera's frames always reach the same worker — its per-camera
   tracker stays correct with no locking. The queue is bounded; an overfull queue means the
   GPU can't keep up, and the dropped frame shows as a `pushed`–`delivered` gap (the GPU
   ceiling is visible, not hidden in a backlog).
4. **Vehicle detection (iGPU).** Each `OVVehicle` worker maps the NV12 frame to BGR, runs the
   640×640 vehicle YOLO on the iGPU, and decodes the `[1,300,6]` NMS output to frame-coordinate
   boxes (COCO class ids: 2=car, 7=truck, …). This is the heavy stage and the throughput wall.
5. **Track + select (CPU, per worker).** A centroid tracker assigns stable `track_id`s, then
   the top-K largest in-ROI vehicles whose plate isn't confirmed are cropped **from the BGR
   frame the worker already decoded** (no separate crop pool / dmabuf hold needed).
6. **Plate detect + OCR (NPU).** A pool of plate-workers runs `OVPlate` then `OVOcr` on the
   crops — both on the **NPU**. They import a pre-compiled NPU blob, so startup is fast and the
   iGPU is left to do only decode + vehicle.
7. **Stabilise (CPU).** The `OcrStabilizer` votes + confirms-and-holds one plate per track.
8. **Publish (CPU pool).** Publisher threads build the **identical** `camera_analytics` JSON
   (schema v5.0, `worker:"traffic_pilotd"`) and write Redis (+ Event Hub if configured).
9. **Live view (CPU, on demand).** A dedicated thread draws boxes + plate text on the stashed
   BGR frame and pushes annotated JPEGs to Redis when an operator is watching.

## 4. Why the work is split this way (the load budget)

The mixed daemon's scarce resource was the iGPU (decode + AIPU resize). Here it is the iGPU
again, but for a harder reason: it now runs the **vehicle detector** as well as decode. So:

- **iGPU: decode + vehicle YOLO** → this is the throughput wall. Measured ≈ **35 ms/frame**
  with 4 OpenVINO workers ⇒ ~116 fps aggregate at 15 cams.
- **NPU: plate-det + OCR** → fully hidden behind the vehicle rate; reads plates cleanly.
- **CPU: orchestration** → never the limit.

This is the expected trade. On 15× `number_plate.mp4`:

| Backend | Vehicle engine | Decode fps | Vehicle fps (agg) | fps/cam | Plate/OCR |
|---------|----------------|-----------:|------------------:|--------:|-----------|
| Axelera hybrid | AIPU | 360 | **180** | 12.0 | AIPU / NPU |
| **OpenVINO-only** | **iGPU** | 297 | **116** | 7.8 | **NPU / NPU** |

The iGPU vehicle model sustains ~65% of the AIPU's vehicle throughput — moving the heavy
detector off a dedicated accelerator is the whole cost. Plate/OCR on the NPU are free either
way. To trade accuracy for speed, swap `VEHICLE_ONNX` (26n ↔ 26s). See the live, regenerable
comparison on the `/bench` page (the collaborative cascading log).

## 5. Threading model (data plane)

Everything is staged through bounded queues; the vehicle workers replace the single AIPU
completion callback with a routed pool:

| Thread(s) | Count | Responsibility |
|-----------|-------|----------------|
| GStreamer streaming (`on_new_sample`) | 1/camera | fps gate → route frame to a vehicle worker |
| **Vehicle workers** (`vehicle_worker`) | `VEHICLE_WORKERS` (4) | OVVehicle on iGPU; map+detect, track, top-K, crop |
| Plate workers (`plate_worker`) | `PLATE_WORKERS` (4) | OVPlate + OVOcr on the **NPU**, feed stabiliser |
| Publisher | 2 | build JSON, write Redis + Event Hub |
| Live-view | 1 | annotated MJPEG on demand |
| Timestamp-OCR | 1 | OCR a burned-in clock ROI (~1 Hz) for cams that configure it |
| Schedule | 1 | re-resolve time-of-day rules every ~15 s |
| GLib main loop | 1 | GStreamer bus/pad events + file-loop seeks |

Frames for a given camera are pinned to one vehicle worker (`sid % N`), which is what lets the
per-camera tracker run lock-free.

## 6. Interfaces & NPU enablement

The Redis/Event-Hub/config contracts are **identical** to the mixed daemon — that is the design
constraint that lets this binary slot under the same UI, analytics consumer, and `/bench`
capture with zero changes:

- **Redis `traffic:analytics`** — `camera_analytics` JSON, schema v5.0, `worker:"traffic_pilotd"`.
- **Redis `live:want:<cam>` / `live:frame:<cam>`** — the on-demand live-view handshake.
- **Config JSON** — the same `config/cameras.json` schema (per-camera URI + levers).
- **Stat log lines** — the per-camera `pushed/skipped/delivered/veh` line and an `OV-callback:`
  line in the same shape the AIPU one uses, so the `/bench` capture parses both backends.

Enabling the **NPU in-process** is the one thing unique to this daemon (handled by
`cpp/run_openvino_daemon.sh`, no manual steps):

1. **Clean `LD_LIBRARY_PATH`** = OpenVINO libs + `/usr/lib/x86_64-linux-gnu` only. Any
   `/opt/axelera/runtime` path leaking in breaks NPU enumeration (the AIPU runtime's bundled
   Level-Zero loader shadows it).
2. **`NPU_COMPILER_TYPE=DRIVER`.** This OpenVINO build ships no standalone MLIR compiler-loader
   `.so`; the in-driver compiler (`libze_intel_npu.so`) is used instead.
3. **Pre-compile the NPU blob once** (single-threaded) before spawning the plate-worker pool,
   so the workers *import* the blob instead of racing to compile+export it. The warm objects are
   intentionally leaked — destroying an NPU-compiled model mid-process corrupts the heap, so the
   daemon (like the mixed one) skips C++ teardown and `_exit(0)`s.

## Build & run

```bash
# build
cmake --build cpp/build --target traffic_pilotd_openvino -j"$(nproc)"

# run the full stack in OpenVINO-only mode (tags runs "all_openvino")
CAMERAS_FILE=$PWD/config/cameras-15-plate.json ./start_openvino.sh
#   vs the Axelera hybrid: CAMERAS_FILE=... ./start.sh
# stop either with the shared ./stop.sh

# device placement is env-tunable without a rebuild:
#   VEHICLE_DEVICE=GPU PLATE_DEVICE=NPU OCR_DEVICE=NPU  (defaults)
#   VEHICLE_ONNX=models/weights/vehicle_yolo26s_640.onnx  (accuracy over speed)
```

New/changed files: `cpp/ov_vehicle.hpp`, `cpp/traffic_pilotd_openvino.cpp`,
`cpp/run_openvino_daemon.sh`, `start_openvino.sh`, `tools/bench/make_cameras.py`, a CMake target,
and an `NPU_COMPILER_TYPE` passthrough in `cpp/ov_infer.hpp`. Nothing in `services/`, `ui/`, or
the Axelera path is modified.
