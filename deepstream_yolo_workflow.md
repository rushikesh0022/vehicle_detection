# DeepStream Documentation: YOLOv3 JPEG Image Inference

Date: June 4, 2026

## 1. Objective

The objective of this experiment was to run object detection on a JPEG image using NVIDIA DeepStream 9.0 inside Docker on WSL2 with GPU acceleration.

The pipeline used a YOLOv3 ONNX model through Triton Inference Server and generated an annotated JPEG image with detected objects.

## 2. Simple Pipeline Flow

```text
JPEG Image
  -> DeepStream GStreamer Pipeline
  -> Triton Inference Server
  -> YOLOv3 ONNX Model
  -> Object Detection
  -> Bounding Box Rendering
  -> Annotated JPEG Output
```

## 3. System Environment

```text
Host OS: Windows 11
Linux Environment: WSL2 Ubuntu
Runtime: Docker Engine
GPU: NVIDIA RTX A4000
DeepStream Container: nvcr.io/nvidia/deepstream:9.0-triton-multiarch
DeepStream SDK: 9.0.0
TensorRT: 10.14
cuDNN: 9.17
```

The DeepStream version was verified with:

```bash
deepstream-app --version-all
```

## 4. Input And Output

Input image on Windows:

```text
C:\Users\Administrator\Downloads\bikes.jpeg
```

Input image inside the container:

```text
/workspace/output/bikes.jpeg
```

Generated output image:

```text
/workspace/output/deepstream_yolo_bikes.jpeg
```

Output location from WSL:

```text
~/deepstream-output/deepstream_yolo_bikes.jpeg
```

## 5. Model Used

```text
Model: YOLOv3-10 ONNX
Triton model name: yolov3-10_onnx
DeepStream sample folder:
/opt/nvidia/deepstream/deepstream-9.0/sources/TritonOnnxYolo
```

ONNX model path:

```text
/opt/nvidia/deepstream/deepstream-9.0/sources/TritonOnnxYolo/triton_model_repo/yolov3-10_onnx/1/model.onnx
```

Custom YOLO post-processing library:

```text
/opt/nvidia/deepstream/deepstream-9.0/sources/TritonOnnxYolo/nvdsinferserver_custom_impl_yolo/libnvdstriton_custom_impl_yolo.so
```

This custom library handles YOLO output decoding, bounding box generation, object class mapping, and non-maximum suppression.

## 6. Pipeline Components

The GStreamer pipeline used these main components:

```text
filesrc
  -> jpegdec
  -> videoconvert
  -> nvvideoconvert
  -> nvstreammux
  -> nvinferserver
  -> nvvideoconvert
  -> nvdsosd
  -> nvvideoconvert
  -> jpegenc
  -> filesink
```

Component purpose:

- `filesrc` reads the input JPEG image.
- `jpegdec` decodes the JPEG image.
- `videoconvert` converts the decoded image to a compatible raw format.
- `nvvideoconvert` moves the frame into NVIDIA GPU memory.
- `nvstreammux` creates a DeepStream batch with `batch-size=1`.
- `nvinferserver` sends the frame to Triton Inference Server.
- `nvdsosd` draws bounding boxes, labels, and confidence scores.
- `jpegenc` encodes the final annotated frame as JPEG.
- `filesink` writes the final image to disk.

## 7. Prepare The Input Image

Create the WSL output folder:

```bash
mkdir -p "$HOME/deepstream-output"
```

Copy the image from Windows into WSL:

```bash
cp /mnt/c/Users/Administrator/Downloads/bikes.jpeg \
  "$HOME/deepstream-output/bikes.jpeg"
```

Verify the image exists:

```bash
ls -lh "$HOME/deepstream-output/bikes.jpeg"
```

## 8. Start The DeepStream Container

Run the DeepStream 9.0 Triton container:

```bash
sudo docker run -it --rm \
  --name deepstream-yolo-image \
  --net=host \
  --gpus all \
  --privileged \
  -e CUDA_CACHE_DISABLE=0 \
  -e NVIDIA_DRIVER_CAPABILITIES=compute,utility,video,graphics \
  -v "$HOME/deepstream-output:/workspace/output" \
  nvcr.io/nvidia/deepstream:9.0-triton-multiarch
```

## 9. Go To The YOLO Sample Directory

Inside the container:

```bash
cd /opt/nvidia/deepstream/deepstream-9.0/sources/TritonOnnxYolo
```

Verify the required files:

```bash
ls -lh /workspace/output/bikes.jpeg
ls -lh triton_model_repo/yolov3-10_onnx/1/model.onnx
ls -lh nvdsinferserver_custom_impl_yolo/libnvdstriton_custom_impl_yolo.so
```

## 10. Run YOLO Inference

Set the config file:

```bash
CONFIG=/opt/nvidia/deepstream/deepstream-9.0/sources/TritonOnnxYolo/config_triton_inferserver_primary_yolov3_onnx_custom.txt
```

Run the pipeline:

```bash
gst-launch-1.0 -e \
  filesrc location=/workspace/output/bikes.jpeg ! \
  jpegdec ! \
  videoconvert ! \
  nvvideoconvert ! \
  "video/x-raw(memory:NVMM),format=NV12,width=640,height=480" ! \
  mux.sink_0 \
  nvstreammux name=mux batch-size=1 width=640 height=480 live-source=0 batched-push-timeout=40000 ! \
  nvinferserver config-file-path=$CONFIG batch-size=1 ! \
  nvvideoconvert ! \
  nvdsosd ! \
  nvvideoconvert ! \
  "video/x-raw,format=I420" ! \
  jpegenc ! \
  filesink location=/workspace/output/deepstream_yolo_bikes.jpeg
```

## 11. Successful Logs

During a successful run, Triton loads the YOLO model and initializes it on the GPU.

Important log lines:

```text
loading: yolov3-10_onnx:1
TRITONBACKEND_ModelInitialize: yolov3-10_onnx
TRITONBACKEND_ModelInstanceInitialize: yolov3-10_onnx_0_0
successfully loaded 'yolov3-10_onnx'
```

Pipeline completion logs:

```text
Pipeline is PREROLLED ...
Setting pipeline to PLAYING ...
Got EOS from element "pipeline0".
EOS received - stopping pipeline...
Setting pipeline to NULL ...
Freeing pipeline ...
```

These logs confirm that the model loaded, inference ran, and the pipeline finished cleanly.

## 12. Verify The Output

Check that the output image was created:

```bash
ls -lh /workspace/output/deepstream_yolo_bikes.jpeg
```

Open the output image from Windows:

```bash
explorer.exe "$(wslpath -w "$HOME/deepstream-output/deepstream_yolo_bikes.jpeg")"
```

## 13. Result

The experiment successfully confirmed that DeepStream 9.0 can process a JPEG image through a YOLOv3 ONNX model using Triton Inference Server inside WSL2 Docker.

The final output was an annotated JPEG image with bounding boxes and labels drawn on detected objects.

## 14. Summary

This setup verified the integration of:

- WSL2
- Docker GPU runtime
- NVIDIA DeepStream 9.0
- Triton Inference Server
- TensorRT
- YOLOv3 ONNX
- GStreamer pipeline
- GPU-accelerated object detection

This document only covers the completed YOLOv3 JPEG inference experiment. It does not go into the full installation or driver setup process.
