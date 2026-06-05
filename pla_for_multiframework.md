# Multi-Backend Reuse Plan

This document explains what can stay the same and what must change if the
edge agent is extended from one inference backend to multiple backends:

- Voyager SDK on Axelera Metis
- OpenVINO on Intel CPU/iGPU/NPU
- DeepStream on NVIDIA GPU/Jetson

The goal is reusability. The edge agent should not be rewritten for every CV
project. Only the inference backend should change, while tracking, analytics,
MQTT, HTTP, and dashboard behavior stay reusable.

## Current Situation

The current edge agent is hardcoded for Voyager.

The startup file creates `VoyagerEngine` directly:

```python
from edge_agent.pipeline.voyager_engine import VoyagerEngine
```

The Voyager engine then calls:

```python
from axelera.app.stream import create_inference_stream
from axelera.app.config import Source
```

So installing OpenVINO or DeepStream alone will not make the current code use
those frameworks. New backend adapter files are required.

## Files That Can Stay As-Is

These files do not care which inference framework is used, as long as every
backend sends detections in the same format.

```text
edge_agent/__init__.py
edge_agent/schemas.py
edge_agent/http_server.py
edge_agent/comms/__init__.py
edge_agent/comms/mqtt_listener.py
edge_agent/comms/mqtt_publisher.py
edge_agent/manager/__init__.py
edge_agent/manager/camera_manager.py
edge_agent/manager/config_store.py
edge_agent/pipeline/__init__.py
edge_agent/pipeline/ocsort.py
edge_agent/pipeline/vehicle_analytics.py
edge_agent/pipeline/video_decoder.py
```

Why these stay the same:

- `ocsort.py` only needs bounding boxes.
- `vehicle_analytics.py` only needs tracked objects and zones.
- `mqtt_publisher.py` publishes final metrics, not raw model output.
- `mqtt_listener.py` receives camera/zone assignment messages.
- `http_server.py` serves snapshots and MJPEG frames from `CameraStream`.
- `video_decoder.py` is still useful for source preflight and fallback
  snapshots, even if a backend does its own decoding.

## Semi Changes Needed

These files mostly stay, but have small hardcoded Voyager/Axelera pieces that
should become generic.

### `edge_agent/config.py`

Before:

```python
VOYAGER_NETWORK = os.environ.get("VOYAGER_NETWORK", "yolov8s-coco")
```

After:

```python
INFERENCE_BACKEND = os.environ.get("INFERENCE_BACKEND", "voyager")

# Same YOLO model concept for all backends.
# Voyager loads it by SDK network name.
# OpenVINO loads the converted XML/BIN files.
# DeepStream loads its config file for the same YOLO model.
YOLO_MODEL = "yolov8s-coco"

VOYAGER_NETWORK = os.environ.get("VOYAGER_NETWORK", YOLO_MODEL)

OPENVINO_MODEL_XML = os.environ.get(
    "OPENVINO_MODEL_XML",
    "models/openvino/yolov8s-coco.xml",
)
OPENVINO_DEVICE = os.environ.get("OPENVINO_DEVICE", "AUTO")

DEEPSTREAM_CONFIG_PATH = os.environ.get(
    "DEEPSTREAM_CONFIG_PATH",
    "models/deepstream/yolov8s-coco/config_infer_primary.txt",
)
```

This lets the same edge agent run with:

```bash
INFERENCE_BACKEND=voyager
INFERENCE_BACKEND=openvino
INFERENCE_BACKEND=deepstream
```

The logical model is still `yolov8s-coco` for all three. The paths are
different because the frameworks load models differently:

```text
Voyager    -> SDK network name: yolov8s-coco
OpenVINO   -> converted model files: yolov8s-coco.xml / yolov8s-coco.bin
DeepStream -> config file that points to ONNX/TensorRT/labels/parser settings
```

Reason:

- Voyager is an SDK framework with registered network names. When we pass
  `yolov8s-coco`, Voyager resolves that name inside the Voyager SDK/model zoo
  and creates the inference stream.
- OpenVINO does not know Voyager network names. It loads model artifacts from
  disk, so it needs a path to the converted OpenVINO IR file, usually
  `yolov8s-coco.xml` with a matching `yolov8s-coco.bin`.
- DeepStream does not load only a simple model name either. It usually needs a
  GStreamer/NVIDIA inference config file. That config points to the model,
  labels, parser, input size, confidence thresholds, and TensorRT engine
  settings.

So all three can represent the same YOLO model idea, but they do not load it in
the same way:

```text
same model idea -> different framework loading method
```

### `edge_agent/main.py`

Before:

```python
def _create_engine():
    """Create VoyagerEngine for Axelera Metis."""
    try:
        from edge_agent.pipeline.voyager_engine import VoyagerEngine
        engine = VoyagerEngine(
            network=cfg.VOYAGER_NETWORK,
            conf=cfg.INFERENCE_CONF,
            iou=cfg.INFERENCE_IOU,
        )
        logger.info("Using VoyagerEngine (Axelera Metis, network=%s)",
                     cfg.VOYAGER_NETWORK)
        return engine
    except Exception as e:
        logger.error("VoyagerEngine creation failed: %s", e)
        raise
```

After:

```python
def _create_engine():
    """Create the configured inference backend."""
    try:
        from edge_agent.pipeline.engine_factory import create_engine

        engine = create_engine()
        logger.info("Using inference backend=%s", cfg.INFERENCE_BACKEND)
        return engine
    except Exception as e:
        logger.error("Inference engine creation failed: %s", e)
        raise
```

Then `engine_factory.py` decides which backend to load.

Important:

```text
main.py does not import Voyager/OpenVINO/DeepStream directly anymore.
main.py only calls create_engine().
engine_factory.py reads INFERENCE_BACKEND and returns the selected engine.
```

So the engine changes here:

```text
INFERENCE_BACKEND=voyager    -> create_engine() returns VoyagerEngine
INFERENCE_BACKEND=openvino   -> create_engine() returns OpenVINOEngine
INFERENCE_BACKEND=deepstream -> create_engine() returns DeepStreamEngine
```

If `INFERENCE_BACKEND` is not set, the default is `voyager` because Voyager is
the current working backend.

### `edge_agent/pipeline/camera_stream.py`

Before:

```python
from edge_agent.pipeline.voyager_engine import VoyagerEngine
self._use_voyager = isinstance(engine, VoyagerEngine)
```

After:

```python
self._use_inference = hasattr(engine, "add_stream")
```

Before:

```python
if self._use_voyager and self._is_local_source():
    ...

self._start_voyager_stream()

logger.info("[%s] Started (source=%s, zones=%d, voyager=%s)",
            self._camera_id, self._source_url,
            len(self._analytics), self._use_voyager)
```

After:

```python
if self._use_inference and self._is_local_source():
    ...

self._start_inference_stream()

logger.info("[%s] Started (source=%s, zones=%d, inference=%s)",
            self._camera_id, self._source_url,
            len(self._analytics), self._use_inference)
```

Before:

```python
def _start_voyager_stream(self) -> None:
    if not self._use_voyager:
        return
    with self._startup_lock:
        if self._voyager_started or not self._running:
            return
        if not cfg.KEEP_FALLBACK_DECODER_OPEN:
            self._fallback_decoder.stop()
        self._engine.add_stream(
            self._camera_id,
            self._source_url,
            self._on_frame,
            error_callback=self.mark_errored,
        )
        self._voyager_started = True
```

After:

```python
def _start_inference_stream(self) -> None:
    if not self._use_inference:
        return
    with self._startup_lock:
        if self._inference_started or not self._running:
            return
        if not cfg.KEEP_FALLBACK_DECODER_OPEN:
            self._fallback_decoder.stop()
        self._engine.add_stream(
            self._camera_id,
            self._source_url,
            self._on_frame,
            error_callback=self.mark_errored,
        )
        self._inference_started = True
```

Rename internal wording:

```text
_use_voyager -> _use_inference
_start_voyager_stream -> _start_inference_stream
_voyager_started -> _inference_started
voyager log labels -> inference log labels
```

The behavior can stay the same.

### `edge_agent/system_metrics.py`

This file is currently Axelera-focused because it checks `axsystemserver` and
`axmonitor`.

It should become backend-aware:

```text
voyager    -> collect Axelera metrics
openvino   -> collect CPU/iGPU/NPU metrics when available
deepstream -> collect NVIDIA GPU metrics when available
```

If a backend does not have telemetry tools available, it should return a note
such as:

```text
backend telemetry unavailable
```

This should not block inference.

### `edge_agent/pipeline/voyager_engine.py`

This file can stay as the Voyager backend. Its public methods become the
pattern that OpenVINO and DeepStream should copy.

Do not remove it. Treat it as:

```text
VoyagerEngine = backend implementation for Axelera Metis
```

## Complete/New Backend Changes Needed

Add these files:

```text
edge_agent/pipeline/engine_factory.py
edge_agent/pipeline/openvino_engine.py
edge_agent/pipeline/deepstream_engine.py
```

No separate `base_engine.py` is required for the first version. Python does not
need it as long as every backend class exposes the same public functions.

Use `VoyagerEngine` as the reference contract. Every backend should implement:

```python
add_stream(cam_id, source_url, callback, error_callback=None)
remove_stream(cam_id)
stop_all()
get_snapshot_frame(cam_id)
avg_inf_ms
avg_inf_fps
```

Every backend must eventually call the saved camera callback:

```python
callback(frame_bgr, detections, frame_w, frame_h)
```

`detections` must always be:

```python
[
    (x1, y1, x2, y2, confidence, class_id),
]
```

This is the most important reusable interface. If this format is preserved,
OC-SORT, zone analytics, MQTT, HTTP snapshots, and the dashboard do not need to
know which backend produced the boxes.

### `engine_factory.py`

Chooses the backend from configuration.

This is the only place that should decide which framework class is loaded for
the first version.

Example:

```python
from edge_agent import config as cfg


def create_engine():
    backend = cfg.INFERENCE_BACKEND.lower()

    if backend == "voyager":
        from edge_agent.pipeline.voyager_engine import VoyagerEngine
        return VoyagerEngine(
            network=cfg.VOYAGER_NETWORK,
            conf=cfg.INFERENCE_CONF,
            iou=cfg.INFERENCE_IOU,
        )

    if backend == "openvino":
        from edge_agent.pipeline.openvino_engine import OpenVINOEngine
        return OpenVINOEngine(
            model_xml=cfg.OPENVINO_MODEL_XML,
            device=cfg.OPENVINO_DEVICE,
            conf=cfg.INFERENCE_CONF,
            iou=cfg.INFERENCE_IOU,
        )

    if backend == "deepstream":
        from edge_agent.pipeline.deepstream_engine import DeepStreamEngine
        return DeepStreamEngine(
            config_path=cfg.DEEPSTREAM_CONFIG_PATH,
            conf=cfg.INFERENCE_CONF,
            iou=cfg.INFERENCE_IOU,
        )

    raise ValueError(f"Unknown inference backend: {cfg.INFERENCE_BACKEND}")
```

### `openvino_engine.py`

This backend should:

1. Open camera/video source.
2. Decode frames, usually with OpenCV or GStreamer.
3. Preprocess frame for the OpenVINO model.
4. Run OpenVINO inference.
5. Postprocess YOLO outputs into pixel boxes.
6. Filter vehicle classes.
7. Run NMS.
8. Call:

```python
callback(frame_bgr, detections, frame_w, frame_h)
```

OpenVINO is the easier second backend because it can be written in normal
Python and can use OpenCV frame reading.

### `deepstream_engine.py`

This backend should:

1. Build a DeepStream/GStreamer pipeline.
2. Decode video inside DeepStream.
3. Run `nvinfer` or a DeepStream-compatible inference plugin.
4. Read metadata from frames.
5. Convert metadata to:

```python
(x1, y1, x2, y2, confidence, class_id)
```

6. Convert frame memory into `frame_bgr` if snapshots/annotations are needed.
7. Call:

```python
callback(frame_bgr, detections, frame_w, frame_h)
```

DeepStream is harder because it wants to own the full video pipeline.

## Backend Selection Design

Use one backend per edge agent.

This matches the deployment model: all cameras assigned to one edge device use
the same edge hardware/framework. Once `INFERENCE_BACKEND` is selected at
startup, the whole edge agent keeps using that backend until the process is
restarted with a different value.

Set one environment variable before starting the edge agent:

```bash
export INFERENCE_BACKEND=voyager
```

or:

```bash
export INFERENCE_BACKEND=openvino
```

or:

```bash
export INFERENCE_BACKEND=deepstream
```

All cameras use the same backend.

Flow:

```text
terminal env var
    -> config.INFERENCE_BACKEND
    -> engine_factory.create_engine()
    -> one selected engine object
    -> CameraManager
    -> all CameraStream objects use that same engine
```

Example:

```bash
export INFERENCE_BACKEND=openvino
python -m edge_agent.main
```

Then `main.py` still runs the same code:

```python
engine = create_engine()
```

But `engine_factory.py` returns `OpenVINOEngine` because the environment
variable is set to `openvino`.

Do not choose backend per camera for this project. Camera requests should only
describe the camera source and zones. Backend choice belongs to the edge agent
environment because it is tied to the hardware/runtime installed on that edge
machine.

## What Changes When A Backend Changes

### Should Not Be Affected

If the backend contract is preserved, these should not be affected:

```text
MQTT publishing
HTTP snapshots/routes
zone analytics
OC-SORT tracking
cloud dashboard
database schema
camera and zone configuration
```

### May Be Affected

```text
decoding
raw snapshot availability
latency/FPS telemetry
hardware telemetry
model output parsing
class ID mapping
```

DeepStream especially may affect decoding because it normally owns the
GStreamer pipeline.

## Reuse Rule For Future CV Projects

For any future CV project, keep this rule:

```text
New model/framework changes only the backend adapter.
The rest of the app receives normalized frames and normalized detections.
```

The normalized detection format should remain:

```python
(x1, y1, x2, y2, confidence, class_id)
```

If a future CV task is not object detection, for example segmentation,
classification, pose, or OCR, create a new normalized output contract. Do not
mix incompatible outputs into the vehicle detection contract.
