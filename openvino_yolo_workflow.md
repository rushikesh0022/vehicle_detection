# OpenVINO YOLO Workflow Validation

Last reviewed: 2026-06-05

This document explains the workflow used to validate YOLO object detection with OpenVINO on an Intel Core Ultra platform. The goal is to confirm that the system can run computer-vision inference on the built-in Intel CPU, Intel Arc GPU, and Intel AI Boost NPU without requiring an external accelerator card.

Target platform from the validation run:

```text
Intel Core Ultra 9 285H
Integrated Intel graphics
Intel AI Boost NPU
Ubuntu Linux
OpenVINO Runtime
Ultralytics YOLO exported to OpenVINO IR
```

The validation question was:

```text
Can this Intel Core Ultra system run the OpenVINO + YOLO detection pipeline by itself?
```

The answer from the validation run was:

```text
CPU inference: PASS
GPU inference: PASS
NPU inference: PASS after NPU driver upgrade
Annotated image generation: PASS
External accelerator required for this test: NO
```

## 1. What This Workflow Proves

This workflow does not try to force CPU, GPU, and NPU to run the same single image at the same time.

Instead, it validates each OpenVINO target device separately:

```text
Image
  |
  v
YOLO OpenVINO model
  |
  v
Choose one target device
  |
  +--> CPU
  +--> GPU
  +--> NPU
  |
  v
Bounding boxes
  |
  v
Annotated output image
```

The purpose is to determine which integrated device is suitable for the computer-vision workload.

In a real surveillance system, the CPU usually handles camera capture, video decode, tracking, alerts, database writes, and UI logic. The GPU or NPU is then used for YOLO inference.

## 2. Hardware Verification

The first step is to confirm Linux can see the Intel GPU and NPU.

### GPU Detection

```bash
lspci -nnk | grep -A4 -Ei "vga|display|3d"
```

Expected signs:

```text
Intel Graphics
Kernel driver in use: i915
```

This confirms the integrated Intel GPU is visible and the `i915` kernel driver is loaded.

### NPU Detection

```bash
lspci -nnk | grep -A4 -Ei "npu|vpu|neural|processing|accel"
```

Expected signs:

```text
Intel NPU / Intel VPU / Intel AI Boost device is visible
Kernel driver in use: intel_vpu
```

Linux may expose the NPU using `vpu` naming because the kernel driver is named `intel_vpu`.

## 3. Device Node Verification

### GPU Render Node

```bash
ls -lah /dev/dri
```

Expected:

```text
renderD128
```

The `/dev/dri/renderD*` node is required for non-display GPU compute access.

### NPU Device Node

```bash
ls -lah /dev/accel/accel0
```

Expected:

```text
crw-rw---- root render ...
```

This confirms the NPU device node exists and is assigned to the `render` group.

If the user is not in the `render` group:

```bash
sudo usermod -a -G render,video "$USER"
newgrp render
```

Verify:

```bash
groups
```

The output should include:

```text
render video
```

## 4. NPU Driver Verification

Check the NPU kernel module:

```bash
modinfo intel_vpu
lsmod | grep intel_vpu
```

Expected:

```text
intel_vpu module exists
intel_vpu is loaded
```

Check kernel logs:

```bash
sudo dmesg | grep -iE "intel_vpu|vpu|npu|accel" | tail -n 80
```

Expected signs:

```text
Intel VPU/NPU initialized
Firmware loaded successfully
```

In the validation run, firmware similar to this was loaded:

```text
intel/vpu/vpu_37xx_v1.bin
```

## 5. OpenVINO Runtime Installation

Create a dedicated workspace and Python virtual environment:

```bash
mkdir -p ~/openvino-work
cd ~/openvino-work

python3 -m venv openvino_env
source openvino_env/bin/activate
```

Upgrade pip:

```bash
python -m pip install --upgrade pip
```

Install OpenVINO:

```bash
python -m pip install openvino
```

## 6. OpenVINO Device Validation

Run:

```bash
source ~/openvino-work/openvino_env/bin/activate
cd ~/openvino-work

python - <<'PY'
from openvino import Core, get_version

core = Core()
print("OpenVINO:", get_version())
print("Available devices:", core.available_devices)
PY
```

Successful full-device output:

```text
['CPU', 'GPU', 'NPU']
```

Meaning:

```text
CPU: OpenVINO base runtime works
GPU: Intel GPU runtime and permissions work
NPU: Intel NPU driver, compiler, firmware, Level Zero stack, and permissions work
```

## 7. Test Image Preparation

Create an image folder:

```bash
mkdir -p ~/openvino-work/images
```

Copy a test image:

```bash
cp ~/Downloads/traffic.jpeg ~/openvino-work/images/test.jpg
```

Verify:

```bash
ls -lh ~/openvino-work/images
```

Expected:

```text
test.jpg
```

## 8. YOLO Export To OpenVINO

Install Ultralytics if needed:

```bash
source ~/openvino-work/openvino_env/bin/activate
python -m pip install ultralytics opencv-python numpy pillow onnx
```

Export YOLO to OpenVINO IR:

```bash
cd ~/openvino-work

yolo export \
  model=yolo26n.pt \
  format=openvino \
  imgsz=640
```

Generated output:

```text
yolo26n_openvino_model/
```

Expected files inside:

```text
model.xml
model.bin
metadata.yaml
```

If `yolo26n.pt` is not available in the installed Ultralytics package, use:

```bash
yolo export \
  model=yolo11n.pt \
  format=openvino \
  imgsz=640
```

Then replace `yolo26n_openvino_model` with `yolo11n_openvino_model` in the commands below.

## 9. CPU Inference Test

Run:

```bash
source ~/openvino-work/openvino_env/bin/activate
cd ~/openvino-work

yolo predict \
  model=yolo26n_openvino_model \
  source=images/test.jpg \
  device=intel:cpu \
  conf=0.25 \
  save=True
```

Validation result from the run:

```text
6 persons
1 car
3 motorcycles
CPU inference succeeded
```

CPU is the baseline. If CPU inference fails, fix the model export or OpenVINO installation before testing GPU or NPU.

## 10. GPU Inference Test

Run:

```bash
yolo predict \
  model=yolo26n_openvino_model \
  source=images/test.jpg \
  device=intel:gpu \
  conf=0.25 \
  save=True
```

Validation result from the run:

```text
6 persons
1 car
3 motorcycles
GPU inference succeeded
```

The Intel Arc GPU is expected to be the strongest built-in candidate for YOLO-style computer-vision workloads on this platform.

## 11. Initial NPU Failure

Initial NPU command:

```bash
yolo predict \
  model=yolo26n_openvino_model \
  source=images/test.jpg \
  device=intel:npu \
  conf=0.25 \
  save=True
```

Observed failure:

```text
IE.Floor op operand must be float
tensor<1x300xsi64>
Failed to create a valid MLIR module
Compilation failed
```

Root cause:

```text
The installed Intel NPU compiler stack was 1.28.0.
That compiler stack could not compile the exported YOLO model because of unsupported operations.
```

This was not a hardware failure. It was also not a basic OpenVINO runtime failure. It was an NPU compiler/model compatibility issue.

## 12. NPU Driver Upgrade

The NPU stack was upgraded to Intel Linux NPU driver `1.33.0`.

Download:

```bash
mkdir -p ~/openvino-work/npu-driver
cd ~/openvino-work/npu-driver

wget https://github.com/intel/linux-npu-driver/releases/download/v1.33.0/linux-npu-driver-v1.33.0.20260529-26625960453-ubuntu2404.tar.gz
tar -xf linux-npu-driver-v1.33.0.20260529-26625960453-ubuntu2404.tar.gz
```

Install:

```bash
sudo dpkg -i *.deb
sudo apt -f install -y
```

Installed package versions:

```text
intel-driver-compiler-npu 1.33.0
intel-fw-npu 1.33.0
intel-level-zero-npu 1.33.0
```

After installation, reboot:

```bash
sudo reboot
```

## 13. Successful NPU Validation

After the NPU driver upgrade:

```bash
source ~/openvino-work/openvino_env/bin/activate
cd ~/openvino-work

yolo predict \
  model=yolo26n_openvino_model \
  source=images/test.jpg \
  device=intel:npu \
  conf=0.25 \
  save=True
```

Validation result from the run:

```text
6 persons
1 car
3 motorcycles
NPU inference succeeded
```

This confirmed that the NPU could compile and execute the YOLO OpenVINO model after upgrading the NPU compiler stack.

## 14. Annotated Image Generation

Use the GPU path for the final annotated image because it is the strongest practical inference candidate from this validation.

```bash
source ~/openvino-work/openvino_env/bin/activate
cd ~/openvino-work

yolo predict \
  model=yolo26n_openvino_model \
  source=images/test.jpg \
  device=intel:gpu \
  conf=0.25 \
  save=True \
  project=outputs \
  name=openvino_yolo_gpu \
  exist_ok=True
```

Output path:

```text
~/openvino-work/outputs/openvino_yolo_gpu/test.jpg
```

Verify:

```bash
ls -lah ~/openvino-work/outputs/openvino_yolo_gpu
```

Open:

```bash
xdg-open ~/openvino-work/outputs/openvino_yolo_gpu/test.jpg
```

The generated image should contain bounding boxes and class labels for detected objects.

## 15. Workflow Explanation

The practical YOLO/OpenVINO workflow is:

```text
Traffic image
  |
  v
Image decode
  |
  v
Resize/preprocess to 640x640
  |
  v
YOLO OpenVINO model
  |
  v
OpenVINO Runtime
  |
  +--> CPU test
  |
  +--> GPU test
  |
  +--> NPU test
  |
  v
YOLO detections
  |
  v
Bounding boxes
  |
  v
Non-max suppression
  |
  v
Annotated image
  |
  v
Saved JPEG
```

When using the Ultralytics command:

```bash
yolo predict model=yolo26n_openvino_model source=images/test.jpg device=intel:gpu save=True
```

the responsibilities are:

```text
Ultralytics:
  image loading
  preprocessing
  YOLO post-processing
  bounding-box drawing
  image saving

OpenVINO:
  model loading
  model compilation
  inference execution on CPU/GPU/NPU
```

So this is:

```text
Ultralytics YOLO pipeline using OpenVINO as the inference backend
```

It is not a pure hand-written OpenVINO pipeline, but it is the correct short workflow for validating YOLO inference and annotated image output.

## 16. Device Roles In A Surveillance Pipeline

A real AI surveillance pipeline would usually look like:

```text
Camera
  |
  v
Frame capture
  |
  v
Decode and resize
  |
  v
YOLO inference
  |
  v
Detection results
  |
  v
Tracking
  |
  v
Alerts, counting, analytics, database, UI
```

Typical assignment:

```text
CPU:
  camera handling
  video decoding
  tracking
  business logic
  database and UI

GPU or NPU:
  YOLO inference
```

For this validation, CPU, GPU, and NPU were tested separately to confirm each OpenVINO target worked.

## 17. Final Result

Successfully validated:

```text
OpenVINO installation: PASS
CPU detection: PASS
GPU detection: PASS
NPU detection: PASS
YOLO OpenVINO export: PASS
CPU inference: PASS
GPU inference: PASS
NPU inference: PASS after driver upgrade
Annotated image generation: PASS
```

Conclusion:

```text
The Intel Core Ultra 9 285H platform successfully executed the OpenVINO + YOLO pipeline on CPU, GPU, and NPU.

The workflow produced an annotated image with bounding boxes.

For this tested model, the integrated Intel platform can run the pipeline without an external AI accelerator.
```
