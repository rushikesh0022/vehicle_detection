# Voyager SDK YOLO Workflow Validation

Last reviewed: 2026-06-05

This document explains the workflow used to validate YOLO object detection with the Axelera Voyager SDK. The goal is to confirm that the Voyager SDK environment can decode a short video, run YOLOv8 COCO inference through the Axelera runtime, and optionally save an augmented output video with detections.

Target workflow:

```text
Voyager SDK
Axelera runtime/hardware
YOLOv8 COCO model
Sample MP4 video from media/
2-second test clip
Annotated/augmented output video
```

The validation question is:

```text
Can the Voyager SDK run YOLO object detection on a video and produce an augmented output?
```

The expected answer after the workflow succeeds:

```text
Voyager SDK import: PASS
Axelera runtime check: PASS
Sample media available: PASS
Video decode test: PASS
YOLOv8 COCO inference: PASS
Augmented output video: PASS if --save-output is available in the installed SDK
```

## 1. What This Workflow Proves

This workflow validates the Voyager YOLO path end to end:

```text
Sample video
  |
  v
Short 2-second test clip
  |
  v
Video decode
  |
  v
Voyager inference.py
  |
  v
YOLOv8 COCO model
  |
  v
Axelera runtime/hardware
  |
  v
Object detections
  |
  v
Augmented output video
```

The SDK hides most of the low-level work inside `./inference.py`, similar to how DeepStream hides most of the pipeline inside GStreamer plugins and how Ultralytics hides most of the OpenVINO pipeline behind `yolo predict`.

## 2. Required Starting Point

Run these commands from inside the `voyager-sdk` directory with the Voyager virtual environment active.

Check current location and virtual environment:

```bash
pwd
echo "$VIRTUAL_ENV"
test -f ./inference.py && echo "OK: inference.py found"
```

Expected:

```text
The current directory is voyager-sdk
$VIRTUAL_ENV is not empty
OK: inference.py found
```

If `inference.py` is missing, you are not in the Voyager SDK root.

## 3. Check Python And Voyager Packages

Run:

```bash
python - <<'PY'
from axelera.app.stream import create_inference_stream
from axelera.app import config, display
import cv2, numpy

print("OK: Axelera Voyager Python packages installed")
print("SDK path:", config.env.framework)
PY
```

Expected:

```text
OK: Axelera Voyager Python packages installed
SDK path: ...
```

This confirms that the Voyager Python package imports correctly from the active environment.

## 4. Check Axelera Hardware And Runtime

Run:

```bash
command -v axdevice
axdevice
```

Expected:

```text
axdevice command exists
Axelera device/runtime information is printed
```

If `axdevice` is missing, the runtime or SDK environment is not ready.

If `axdevice` exists but does not show a valid device, fix the Axelera hardware/runtime setup before running YOLO inference.

## 5. Check Decode And Encode Tools

Run:

```bash
command -v ffmpeg
command -v ffprobe
command -v gst-launch-1.0
```

Then check required GStreamer plugins:

```bash
gst-inspect-1.0 decodebin
gst-inspect-1.0 x264enc
gst-inspect-1.0 mp4mux
```

Expected:

```text
ffmpeg exists
ffprobe exists
gst-launch-1.0 exists
decodebin exists
x264enc exists
mp4mux exists
```

These tools are used to create the short test clip, test video decode, and save or inspect the output video.

## 6. Install Sample Media If Missing

Voyager SDK sample videos live in the `media/` folder.

Check:

```bash
ls media/*.mp4
```

If sample media is missing, run:

```bash
VENV="$VIRTUAL_ENV"
deactivate
./install.sh --media
source "$VENV/bin/activate"
```

Then verify again:

```bash
ls media/*.mp4
```

## 7. Pick A Sample Video

Use the built-in traffic video when available:

```bash
export SOURCE_VIDEO=media/traffic1_1080p.mp4
test -f "$SOURCE_VIDEO" || export SOURCE_VIDEO="$(find media -type f -iname '*.mp4' | head -n 1)"

echo "$SOURCE_VIDEO"
```

Expected:

```text
media/traffic1_1080p.mp4
```

or another MP4 file from the `media/` directory.

## 8. Create A 2-Second Test Clip

Create an output directory:

```bash
mkdir -p outputs
```

Generate a short clip:

```bash
ffmpeg -y -i "$SOURCE_VIDEO" -t 2 \
  -an -c:v libx264 -preset ultrafast -pix_fmt yuv420p \
  outputs/input_2sec.mp4
```

Verify:

```bash
ls -lh outputs/input_2sec.mp4
ffprobe outputs/input_2sec.mp4
```

This keeps the first test small and easy to debug.

## 9. Decode-Only Test

Before running AI inference, confirm the video can be decoded:

```bash
gst-launch-1.0 -q \
  filesrc location=outputs/input_2sec.mp4 \
  ! decodebin \
  ! videoconvert \
  ! fakesink sync=false
```

Expected:

```text
The command exits successfully without decode errors.
```

If this fails, fix GStreamer/video decoding before testing the Voyager YOLO model.

## 10. Run YOLOv8 COCO Inference

Run the Voyager SDK inference script:

```bash
./inference.py yolov8s-coco-onnx outputs/input_2sec.mp4 \
  --no-display \
  --show-stats
```

Expected:

```text
The model loads.
The video runs through inference.
Stats are printed.
No display window is required.
```

If `yolov8s-coco-onnx` fails because the model name is different in the installed SDK, try:

```bash
./inference.py yolov8s-coco outputs/input_2sec.mp4 \
  --no-display \
  --show-stats
```

The model names checked for this workflow are:

```text
yolov8s-coco-onnx
yolov8s-coco
```

## 11. Check Whether Output Saving Is Supported

Different Voyager SDK versions may expose different save flags. Check help first:

```bash
./inference.py --help | grep -i "save"
```

If the help output contains `--save-output`, use the save-output workflow in the next section.

If it does not, the installed SDK may still support display/stat-only inference but not direct output video writing through that flag.

## 12. Save Augmented Output Video

If `--save-output` is available, run:

```bash
./inference.py yolov8s-coco-onnx outputs/input_2sec.mp4 \
  --save-output outputs/yolo_augmented_2sec.mp4 \
  --show-stats
```

If the ONNX model name fails, use:

```bash
./inference.py yolov8s-coco outputs/input_2sec.mp4 \
  --save-output outputs/yolo_augmented_2sec.mp4 \
  --show-stats
```

Verify:

```bash
ls -lh outputs/yolo_augmented_2sec.mp4
ffprobe outputs/yolo_augmented_2sec.mp4
```

Expected output:

```text
outputs/yolo_augmented_2sec.mp4
```

This video should contain the original frames augmented with YOLO detection overlays if the SDK save path is working.

## 13. Workflow Explanation

The practical Voyager YOLO workflow is:

```text
media/traffic1_1080p.mp4
  |
  v
ffmpeg creates outputs/input_2sec.mp4
  |
  v
GStreamer decode test validates the clip
  |
  v
./inference.py starts Voyager pipeline
  |
  v
YOLOv8 COCO model is selected
  |
  v
Axelera runtime executes inference
  |
  v
Detection metadata is generated
  |
  v
Voyager overlays detections
  |
  v
outputs/yolo_augmented_2sec.mp4
```

Component responsibilities:

```text
ffmpeg:
  creates a short test clip

GStreamer:
  validates video decode path

Voyager inference.py:
  loads the model
  creates the inference stream
  handles video input
  runs inference
  shows stats
  optionally saves augmented output

Axelera runtime/hardware:
  executes the neural-network inference workload
```

## 14. Comparison With DeepStream And OpenVINO

This Voyager workflow is conceptually similar to the DeepStream and OpenVINO workflows:

```text
DeepStream:
  gst-launch pipeline + nvinferserver + nvdsosd

OpenVINO:
  yolo predict + OpenVINO backend

Voyager:
  ./inference.py + Voyager runtime + Axelera hardware
```

The short Voyager command works because the SDK already contains the pipeline logic:

```bash
./inference.py yolov8s-coco-onnx outputs/input_2sec.mp4 --no-display --show-stats
```

That command hides the decode, model loading, inference stream setup, detection post-processing, and stats reporting inside the Voyager SDK.

## 15. Final Checklist

Record the validation result:

```text
Working directory is voyager-sdk: PASS/FAIL
Virtual environment active: PASS/FAIL
inference.py found: PASS/FAIL
Voyager Python imports: PASS/FAIL
axdevice runtime check: PASS/FAIL
ffmpeg/ffprobe available: PASS/FAIL
GStreamer decode plugins available: PASS/FAIL
Sample media available: PASS/FAIL
2-second clip created: PASS/FAIL
Decode-only test: PASS/FAIL
YOLO inference with yolov8s-coco-onnx: PASS/FAIL
YOLO inference fallback yolov8s-coco: PASS/FAIL/NOT NEEDED
--save-output available: PASS/FAIL
Augmented output video created: PASS/FAIL
```

Final expected result:

```text
The Voyager SDK can run YOLOv8 COCO inference on a short video clip and, when supported by the installed SDK, save an augmented MP4 output.
```

## Source Links

- Voyager SDK GitHub: https://github.com/axelera-ai-hub/voyager-sdk
- Axelera Model Zoo: https://docs.axelera.ai/sdk/reference/models/model-zoo/
- Axelera Video Sources: https://docs.axelera.ai/sdk/tutorials/video-sources/
- Axelera Run Inference In Python: https://docs.axelera.ai/sdk/tutorials/run-inference-in-python/
