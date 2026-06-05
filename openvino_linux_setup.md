# OpenVINO Linux Setup Guide

Last reviewed: 2026-06-04

This guide explains the Linux setup and validation flow for testing whether an integrated Intel AI PC platform can carry the computer-vision workload without a separate accelerator card.

Primary target:

```text
Intel Core Ultra 7 255H
Integrated only: CPU + Intel Arc 140T iGPU + Intel AI Boost NPU
Official peak AI rating: 96 overall INT8 TOPS
Official split: 74 GPU INT8 TOPS + 13 NPU INT8 TOPS
```

The practical question is:

```text
Can the Intel CPU + iGPU + NPU run the OpenVINO CV pipeline well enough without an add-in GPU, Hailo, Coral, or other accelerator?
```

The answer should come from checks and measurements, not assumptions. By the end, we should know:

- Whether Linux sees the CPU, GPU, and NPU.
- Whether the required GPU and NPU drivers are installed.
- Whether OpenVINO sees `CPU`, `GPU`, and `NPU`.
- Whether YOLO can run on a captured image and save bounding boxes.
- Whether CPU, GPU, and NPU benchmark numbers are good enough for the project.

## Target Result

The main OpenVINO device check should print something like:

```bash
python -c "from openvino import Core, get_version; c=Core(); print(get_version()); print(c.available_devices)"
```

Expected examples:

```text
['CPU']
['CPU', 'GPU']
['CPU', 'GPU', 'NPU']
```

`CPU` confirms base OpenVINO is working. `GPU` confirms the Intel graphics runtime is available. `NPU` confirms the Intel NPU driver and OpenVINO NPU plugin are available.

For this project, the desired target is:

```text
CPU + GPU + NPU visible to OpenVINO
YOLO image inference works
Annotated output image is saved
Benchmark numbers are recorded for CPU, GPU, and NPU where supported
```

## 0. Recommended Linux Baseline

Use native Linux, not WSL, for the first NPU bring-up.

Recommended:

```text
Ubuntu 24.04 LTS
Kernel 6.8 or newer minimum for OpenVINO Linux support
Kernel 6.17 or newer preferred for latest Intel NPU driver validation
Python 3.9 through 3.12
OpenVINO 2026.2 for this hardware test
```

Ubuntu 22.04 can work, but NPU inference currently needs at least kernel 6.6, and the newest Intel Core Ultra Series 2 systems are easier to validate on Ubuntu 24.04.

OpenVINO 2026.2 is a development release, not a long-term-support release. Use it for this bring-up because current NPU support and Intel's latest reviewed Linux NPU driver are aligned with it. If the project later needs a maintenance/LTS-style deployment baseline, evaluate that separately after the hardware proof is done.

Check OS and kernel:

```bash
cat /etc/os-release
uname -a
uname -r
python3 --version
```

Decision rule:

- If Ubuntu is older than 22.04, upgrade before continuing.
- If kernel is older than 6.6, do not expect NPU inference to work.
- If the platform is Core Ultra Series 2, prefer Ubuntu 24.04 with the newest available kernel from the hardware vendor or Ubuntu HWE/OEM path.

## 1. Hardware Preflight

Run these before installing anything. Save the output in the project notes because these commands become the hardware proof.

### Check CPU Identity

```bash
lscpu | sed -n '1,40p'
grep -m1 "model name" /proc/cpuinfo
```

For the target machine, expect a model string similar to:

```text
Intel(R) Core(TM) Ultra 7 255H
```

Official reference values for the Core Ultra 7 255H:

```text
CPU cores: 16 total
Performance cores: 6
Efficient cores: 8
Low power efficient cores: 2
Threads: 16
Overall peak TOPS INT8: 96
GPU: Intel Arc 140T GPU, 74 peak INT8 TOPS
NPU: Intel AI Boost, 13 peak INT8 TOPS
```

If the device is the next version of this platform, record the same values from Intel ARK or the vendor spec sheet and continue.

### Check PCI Devices

```bash
lspci -nn | grep -Ei "vga|display|3d|npu|vpu|neural|processing|accel"
lspci -nnk | grep -A4 -Ei "vga|display|3d|npu|vpu|neural|processing|accel"
```

Expected:

- Intel graphics device is visible.
- NPU may appear as an NPU, VPU, multimedia, or accelerator-style PCI device depending on kernel and platform naming.

Linux NPU naming note:

```text
Intel's Linux kernel driver still uses the older VPU naming internally.
The kernel module is intel_vpu.
The userspace device is usually /dev/accel/accel0.
```

### Check GPU Device Nodes

```bash
ls -lah /dev/dri || true
ls -lah /dev/dri/renderD* || true
groups
```

Expected:

```text
/dev/dri/renderD*
```

The current user should be in the `render` group for non-root GPU access.

If not:

```bash
sudo usermod -a -G render "$USER"
newgrp render
```

If `newgrp render` is not enough, log out and log back in.

### Check NPU Kernel Driver

```bash
modinfo intel_vpu || true
lsmod | grep intel_vpu || true
sudo modprobe intel_vpu || true
sudo dmesg | grep -iE "intel_vpu|vpu|npu|accel" | tail -n 80
ls -lah /dev/accel || true
ls -lah /dev/accel/accel0 || true
```

Good signs:

```text
intel_vpu module exists
intel_vpu module loads
/dev/accel/accel0 exists
dmesg contains an initialized intel_vpu line
```

OpenVINO's NPU docs show a successful boot message in this shape:

```text
[drm] Initialized intel_vpu 0.<version> for 0000:00:0b.0 on minor 0
```

If `/dev/accel/accel0` exists but the user cannot access it:

```bash
groups
sudo usermod -a -G render "$USER"
newgrp render
ls -lah /dev/accel/accel0
```

If permissions are still wrong, an administrator can temporarily repair them:

```bash
sudo chown root:render /dev/accel/accel0
sudo chmod g+rw /dev/accel/accel0
ls -lah /dev/accel/accel0
```

## 2. Check Existing Driver And OpenVINO Packages

Before installing, check what is already present:

```bash
dpkg -l | grep -Ei "openvino|intel-opencl|intel-level-zero|level-zero|libze|intel-fw-npu|intel-driver-compiler-npu|intel-level-zero-npu" || true
ldconfig -p | grep -Ei "libOpenCL|libze_loader|libze_intel_npu|libnpu_driver_compiler|openvino" || true
which benchmark_app || true
python3 -m pip show openvino || true
```

If OpenVINO might already be installed:

```bash
python3 - <<'PY'
try:
    from openvino import Core, get_version
    core = Core()
    print("OpenVINO:", get_version())
    print("Devices:", core.available_devices)
except Exception as exc:
    print("OpenVINO import failed:", repr(exc))
PY
```

Decision rule:

- If this prints `CPU`, base OpenVINO works.
- If it prints `GPU`, GPU plugin and driver access work.
- If it prints `NPU`, NPU plugin and driver access work.
- If OpenVINO import fails, install OpenVINO.
- If `GPU` or `NPU` is missing, fix drivers before blaming OpenVINO.

## 3. Install Common Linux Prerequisites

```bash
sudo apt update
sudo apt install -y \
  build-essential \
  cmake \
  curl \
  git \
  gpg \
  pciutils \
  python3 \
  python3-pip \
  python3-venv \
  usbutils \
  wget \
  clinfo \
  vainfo \
  v4l-utils \
  ffmpeg
```

Optional but useful for GPU diagnostics:

```bash
sudo apt install -y intel-gpu-tools
```

Make sure the user can access render devices:

```bash
sudo usermod -a -G render,video "$USER"
newgrp render
```

If device access still fails later, reboot once:

```bash
sudo reboot
```

## 4. Install Or Repair Intel GPU Runtime

Skip this only if CPU-only inference is acceptable.

OpenVINO GPU inference requires Intel graphics runtime packages. On Ubuntu 22.04 and 24.04, start with:

```bash
sudo apt update
sudo apt install -y \
  ocl-icd-libopencl1 \
  intel-opencl-icd \
  intel-level-zero-gpu \
  level-zero
sudo usermod -a -G render "$USER"
newgrp render
```

Verify OpenCL and Level Zero visibility:

```bash
clinfo | grep -Ei "platform name|device name|device version|opencl" | head -n 80
ls -lah /dev/dri/renderD*
ldconfig -p | grep -Ei "libOpenCL|libze_loader"
```

Good signs:

```text
clinfo lists Intel GPU or Intel Graphics
/dev/dri/renderD* exists
libOpenCL.so and libze_loader.so are visible
```

If the GPU still does not appear, use Intel's current graphics runtime installation guide or the Intel Graphics Compute Runtime release packages, then repeat the checks. New Core Ultra platforms sometimes need a newer graphics stack than the distro installed by default.

## 5. Install Or Repair Intel NPU Driver

Skip this only if NPU testing is not required.

OpenVINO does not install the Linux NPU kernel and userspace driver for you. The NPU driver stack comes from Intel's Linux NPU driver project.

The NPU stack includes:

```text
intel_vpu kernel module
intel-fw-npu firmware package
intel-level-zero-npu userspace driver
intel-driver-compiler-npu compiler package
Level Zero loader, usually libze_loader.so
```

### NPU Driver Requirements

Minimum:

```text
Ubuntu 22.04 or newer
Linux kernel 6.6 or newer for NPU inference
make, gcc, and kernel headers if building components
render or video group access for the user
```

Recommended for this Core Ultra 7 255H validation:

```text
Ubuntu 24.04 LTS
Latest kernel available for the platform
Latest Intel linux-npu-driver release
OpenVINO 2026.2
```

### Example: Install Latest Reviewed NPU Driver

As of this guide's review date, Intel's latest Linux NPU driver release is `v1.33.0`, verified by Intel with OpenVINO 2026.2 on Ubuntu 24.04.

Always check the release page first. If a newer release exists, use the newer package name.

```bash
mkdir -p ~/openvino-work/npu-driver
cd ~/openvino-work/npu-driver

wget https://github.com/intel/linux-npu-driver/releases/download/v1.33.0/linux-npu-driver-v1.33.0.20260529-26625960453-ubuntu2404.tar.gz
tar -xf linux-npu-driver-v1.33.0.20260529-26625960453-ubuntu2404.tar.gz

sudo apt update
sudo apt install -y libtbb12

sudo dpkg -i *.deb
sudo apt -f install -y
```

The `v1.33.0` release also lists `libze1` 1.27.0 as the Level Zero loader used in its verified configuration. Install the matching package if your system does not already have a compatible `libze1`:

```bash
cd ~/openvino-work/npu-driver
wget https://snapshot.ppa.launchpadcontent.net/kobuk-team/intel-graphics/ubuntu/20260324T100000Z/pool/main/l/level-zero-loader/libze1_1.27.0-1~24.04~ppa2_amd64.deb
sudo dpkg -i libze1_*.deb
sudo apt -f install -y
```

If there is a Level Zero package conflict:

```bash
sudo dpkg --purge --force-remove-reinstreq level-zero level-zero-devel
sudo dpkg -i libze1_*.deb
sudo apt -f install -y
```

Add the user to the render group:

```bash
sudo gpasswd -a "$USER" render
newgrp render
```

Load and verify:

```bash
sudo modprobe intel_vpu
ls -lah /dev/accel/accel0
sudo dmesg | grep -iE "intel_vpu|vpu|npu|accel" | tail -n 80
ldconfig -p | grep -Ei "libze_intel_npu|libnpu_driver_compiler|libze_loader"
```

Expected:

```text
/dev/accel/accel0
libze_intel_npu.so
libnpu_driver_compiler.so
libze_loader.so
```

## 6. Install OpenVINO With Python

This is the recommended path for our project validation because it gives the Python API, `benchmark_app`, and a fast YOLO experiment loop.

```bash
mkdir -p ~/openvino-work
cd ~/openvino-work

python3 -m venv openvino_env
source openvino_env/bin/activate

python -m pip install --upgrade pip
python -m pip install openvino
```

For a reproducible Core Ultra 255H NPU test, pin the version used in the test report:

```bash
python -m pip install "openvino==2026.2.0"
```

If that exact patch version is not available on PyPI, use the newest available `2026.2` package and record:

```bash
python -c "from openvino import get_version; print(get_version())"
```

Verify:

```bash
python - <<'PY'
from openvino import Core, get_version
core = Core()
print("OpenVINO:", get_version())
print("Available devices:", core.available_devices)
for device in core.available_devices:
    print()
    print("Device:", device)
    for key in ("FULL_DEVICE_NAME", "OPTIMIZATION_CAPABILITIES"):
        try:
            print(f"{key}:", core.get_property(device, key))
        except Exception as exc:
            print(f"{key}: unavailable ({exc})")
PY
```

Expected for the full target:

```text
Available devices: ['CPU', 'GPU', 'NPU']
```

## 7. Optional: Install OpenVINO With APT

Use APT instead of PyPI if you need system-wide C/C++ libraries or included sample files.

Add Intel's OpenVINO APT repository:

```bash
cd ~/openvino-work
wget https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB
sudo gpg --output /etc/apt/trusted.gpg.d/intel.gpg --dearmor GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB
```

For Ubuntu 24.04:

```bash
echo "deb https://apt.repos.intel.com/openvino ubuntu24 main" | sudo tee /etc/apt/sources.list.d/intel-openvino.list
```

For Ubuntu 22.04:

```bash
echo "deb https://apt.repos.intel.com/openvino ubuntu22 main" | sudo tee /etc/apt/sources.list.d/intel-openvino.list
```

Then install:

```bash
sudo apt update
apt-cache search openvino | head -n 40
sudo apt install -y openvino
apt list --installed | grep openvino
```

Run the included Python device sample:

```bash
python3 /usr/share/openvino/samples/python/hello_query_device/hello_query_device.py
```

For this repo's Python experiments, still prefer the virtual environment path unless you specifically need system packages.

## 8. Verify OpenVINO Device Access

Activate the environment first:

```bash
source ~/openvino-work/openvino_env/bin/activate
```

Run:

```bash
python - <<'PY'
from openvino import Core, get_version
core = Core()
print("OpenVINO:", get_version())
print("Devices:", core.available_devices)

required = {"CPU", "GPU", "NPU"}
missing = sorted(required - set(core.available_devices))
if missing:
    print("Missing:", missing)
else:
    print("All target devices are visible.")
PY
```

Device interpretation:

- `CPU` missing: OpenVINO install is broken.
- `GPU` missing: graphics runtime or render permissions are not ready.
- `NPU` missing: NPU driver, firmware, compiler, Level Zero, kernel, or permissions are not ready.

Important NPU behavior:

```text
OpenVINO AUTO does not currently include NPU in its default priority list.
Use NPU explicitly when testing NPU.
```

Examples:

```python
core.compile_model(model, "CPU")
core.compile_model(model, "GPU")
core.compile_model(model, "NPU")
```

## 9. Capture A Test Image

If using a USB or built-in camera:

```bash
v4l2-ctl --list-devices
v4l2-ctl --list-formats-ext -d /dev/video0
```

Capture one image:

```bash
mkdir -p ~/openvino-work/images
ffmpeg -y -f v4l2 -video_size 1280x720 -i /dev/video0 -frames:v 1 ~/openvino-work/images/test.jpg
file ~/openvino-work/images/test.jpg
```

If the camera uses a different device, replace `/dev/video0`.

If there is no camera, use any local image:

```bash
mkdir -p ~/openvino-work/images
cp /path/to/your/image.jpg ~/openvino-work/images/test.jpg
```

Check that Python can decode it:

```bash
source ~/openvino-work/openvino_env/bin/activate
python -m pip install opencv-python
python - <<'PY'
import cv2
from pathlib import Path

path = Path.home() / "openvino-work" / "images" / "test.jpg"
img = cv2.imread(str(path))
print("Path:", path)
print("Decoded:", img is not None)
if img is not None:
    print("Shape:", img.shape)
PY
```

## 10. Install YOLO Tooling

Activate the OpenVINO environment:

```bash
source ~/openvino-work/openvino_env/bin/activate
python -m pip install --upgrade pip
python -m pip install ultralytics opencv-python pillow numpy onnx
```

Verify:

```bash
yolo checks
python - <<'PY'
from ultralytics import YOLO
print("Ultralytics import OK")
PY
```

## 11. Export YOLO To OpenVINO

Use the current small model first so we validate the pipeline quickly.

Current Ultralytics docs use YOLO26 examples:

```bash
source ~/openvino-work/openvino_env/bin/activate
cd ~/openvino-work

yolo export model=yolo26n.pt format=openvino imgsz=640
```

If the installed Ultralytics package cannot resolve `yolo26n.pt`, use `yolo11n.pt` first and then upgrade Ultralytics later:

```bash
yolo export model=yolo11n.pt format=openvino imgsz=640
```

This should create:

```text
yolo26n_openvino_model/
```

If you exported `yolo11n.pt`, replace `yolo26n_openvino_model` with `yolo11n_openvino_model` in the commands below.

If the project already uses YOLOv8 or YOLO11, use the same pattern:

```bash
yolo export model=yolov8n.pt format=openvino imgsz=640
yolo export model=yolo11n.pt format=openvino imgsz=640
```

For NPU, keep the first test simple:

```text
static image size
batch 1
dynamic=False
no custom post-processing until basic compile works
```

If the model has trouble compiling on NPU, try an INT8 export with calibration data:

```bash
yolo export model=yolo26n.pt format=openvino imgsz=640 int8=True data=coco8.yaml
```

For a real project model, replace `coco8.yaml` with the project's calibration dataset YAML.

## 12. Run YOLO On One Image And Save Boxes

CPU baseline:

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

GPU test:

```bash
yolo predict \
  model=yolo26n_openvino_model \
  source=images/test.jpg \
  device=intel:gpu \
  conf=0.25 \
  save=True
```

NPU test:

```bash
yolo predict \
  model=yolo26n_openvino_model \
  source=images/test.jpg \
  device=intel:npu \
  conf=0.25 \
  save=True
```

Expected output:

```text
runs/detect/predict*/test.jpg
```

That saved image should contain bounding boxes if the model detects objects.

If CLI paths are awkward, use Python:

```bash
python - <<'PY'
from pathlib import Path
from ultralytics import YOLO

image = Path("images/test.jpg")
model = YOLO("yolo26n_openvino_model")

for device in ("intel:cpu", "intel:gpu", "intel:npu"):
    print(f"\nRunning {device}")
    try:
        result = model.predict(
            source=str(image),
            device=device,
            conf=0.25,
            save=True,
            verbose=False,
        )[0]
        print("Boxes:", len(result.boxes))
        if len(result.boxes):
            print(result.boxes.data.cpu().numpy())
    except Exception as exc:
        print(f"{device} failed: {exc}")
PY
```

Decision rule:

- CPU must pass first.
- GPU should pass if `GPU` appears in OpenVINO devices.
- NPU should be tested explicitly, but some model/export combinations may need static shape or INT8 export.

## 13. Benchmark Raw OpenVINO Inference

`benchmark_app` is installed with the OpenVINO PyPI package. It measures model inference repeatedly. It is good for device comparison because it reports latency and throughput.

Find the OpenVINO model XML:

```bash
find ~/openvino-work -name "*.xml" | sort
```

Set the model path:

```bash
cd ~/openvino-work
MODEL_XML="$(find yolo26n_openvino_model -name '*.xml' | head -n 1)"
echo "$MODEL_XML"
```

CPU latency:

```bash
benchmark_app -m "$MODEL_XML" -d CPU -hint latency -t 60 -i images/test.jpg
```

CPU throughput:

```bash
benchmark_app -m "$MODEL_XML" -d CPU -hint throughput -t 60 -i images/test.jpg
```

GPU latency:

```bash
benchmark_app -m "$MODEL_XML" -d GPU -hint latency -t 60 -i images/test.jpg
```

GPU throughput:

```bash
benchmark_app -m "$MODEL_XML" -d GPU -hint throughput -t 60 -i images/test.jpg
```

NPU latency:

```bash
benchmark_app -m "$MODEL_XML" -d NPU -hint latency -t 60 -i images/test.jpg
```

NPU throughput:

```bash
benchmark_app -m "$MODEL_XML" -d NPU -hint throughput -t 60 -i images/test.jpg
```

If the current `benchmark_app` build rejects `NPU`, record that as a tool limitation and use the end-to-end YOLO benchmark in the next section for the NPU comparison.

Save reports:

```bash
mkdir -p benchmark_reports

benchmark_app -m "$MODEL_XML" -d CPU -hint latency -t 60 -i images/test.jpg \
  -report_type no_counters -report_folder benchmark_reports/cpu_latency

benchmark_app -m "$MODEL_XML" -d GPU -hint latency -t 60 -i images/test.jpg \
  -report_type no_counters -report_folder benchmark_reports/gpu_latency

benchmark_app -m "$MODEL_XML" -d NPU -hint latency -t 60 -i images/test.jpg \
  -report_type no_counters -report_folder benchmark_reports/npu_latency
```

Notes:

- `benchmark_app` repeats the same input, so a single image is enough for a first performance check.
- For production-like numbers, use a folder of representative images.
- `benchmark_app` focuses on model inference. It does not prove the whole application pipeline, camera capture, JPEG decode, tracking, database writes, or UI cost.

## 14. Benchmark End-To-End YOLO Pipeline

Use this when you want to include image decode, preprocessing, OpenVINO inference, YOLO post-processing, and box output overhead.

```bash
source ~/openvino-work/openvino_env/bin/activate
cd ~/openvino-work

python - <<'PY'
import statistics
import time
from ultralytics import YOLO

image = "images/test.jpg"
model = YOLO("yolo26n_openvino_model")
devices = ["intel:cpu", "intel:gpu", "intel:npu"]
warmup = 5
runs = 50

for device in devices:
    print(f"\nDevice: {device}")
    try:
        for _ in range(warmup):
            model.predict(image, device=device, conf=0.25, verbose=False)

        timings = []
        box_counts = []
        for _ in range(runs):
            start = time.perf_counter()
            result = model.predict(image, device=device, conf=0.25, verbose=False)[0]
            elapsed = time.perf_counter() - start
            timings.append(elapsed * 1000)
            box_counts.append(len(result.boxes))

        avg_ms = statistics.mean(timings)
        p50_ms = statistics.median(timings)
        p95_ms = sorted(timings)[int(0.95 * len(timings)) - 1]
        fps = 1000.0 / avg_ms

        print(f"runs={runs}")
        print(f"avg_ms={avg_ms:.2f}")
        print(f"p50_ms={p50_ms:.2f}")
        print(f"p95_ms={p95_ms:.2f}")
        print(f"fps={fps:.2f}")
        print(f"last_box_count={box_counts[-1]}")
    except Exception as exc:
        print(f"FAILED: {exc}")
PY
```

Use these numbers to compare:

```text
CPU: reliable baseline
GPU: likely best integrated-device throughput for CV on Core Ultra 7 255H
NPU: useful if it compiles and gives acceptable latency or better power behavior
```

Because the 255H has 74 GPU INT8 TOPS and 13 NPU INT8 TOPS, do not assume the NPU will beat the GPU for YOLO. Measure both.

## 15. Troubleshooting Matrix

### OpenVINO Import Fails

Check:

```bash
source ~/openvino-work/openvino_env/bin/activate
python --version
python -m pip show openvino
python -c "import openvino; print(openvino.__file__)"
```

Fix:

```bash
python -m pip install --upgrade pip
python -m pip install --force-reinstall openvino
```

### `CPU` Missing From OpenVINO

This usually means the OpenVINO install is broken.

```bash
python -c "from openvino import Core; print(Core().available_devices)"
```

Reinstall OpenVINO in a clean virtual environment.

### `GPU` Missing From OpenVINO

Check:

```bash
ls -lah /dev/dri/renderD*
groups
clinfo | head -n 80
dpkg -l | grep -Ei "intel-opencl|intel-level-zero|level-zero|ocl-icd"
```

Fix:

```bash
sudo apt install -y ocl-icd-libopencl1 intel-opencl-icd intel-level-zero-gpu level-zero
sudo usermod -a -G render "$USER"
newgrp render
```

Then log out and back in or reboot.

### `NPU` Missing From OpenVINO

Check:

```bash
uname -r
modinfo intel_vpu || true
lsmod | grep intel_vpu || true
ls -lah /dev/accel /dev/accel/accel0 || true
sudo dmesg | grep -iE "intel_vpu|vpu|npu|accel" | tail -n 100
dpkg -l | grep -Ei "intel-fw-npu|intel-level-zero-npu|intel-driver-compiler-npu|libze"
ldconfig -p | grep -Ei "libze_intel_npu|libnpu_driver_compiler|libze_loader"
groups
```

Fix path:

1. Upgrade to a kernel supported by the NPU driver.
2. Install the newest Intel Linux NPU driver release.
3. Make sure `/dev/accel/accel0` exists.
4. Make sure the user is in `render` or `video`.
5. Confirm `libze_intel_npu.so`, `libnpu_driver_compiler.so`, and `libze_loader.so` are visible.
6. Reboot after driver or group changes.

### NPU Appears But YOLO Fails

Try:

```bash
yolo export model=yolo26n.pt format=openvino imgsz=640 int8=True data=coco8.yaml
yolo predict model=yolo26n_openvino_model source=images/test.jpg device=intel:npu conf=0.25 save=True
```

Also try a smaller or simpler model first:

```bash
yolo export model=yolo26n.pt format=openvino imgsz=320 int8=True data=coco8.yaml
```

NPU notes:

- Use static shapes first.
- Use batch 1 first.
- Use explicit `device=intel:npu`.
- If `AUTO` is used, remember that NPU is excluded from OpenVINO's default AUTO priority list.

## 16. Acceptance Checklist

Record results in a small table:

```text
OS:
Kernel:
CPU model:
OpenVINO version:
Python version:
GPU runtime packages:
NPU driver version:
OpenVINO devices:
YOLO model:
Image source:
CPU YOLO result:
GPU YOLO result:
NPU YOLO result:
CPU benchmark avg latency:
GPU benchmark avg latency:
NPU benchmark avg latency:
CPU throughput:
GPU throughput:
NPU throughput:
Final decision:
```

Pass criteria for this project:

- `Core().available_devices` contains `CPU`, `GPU`, and ideally `NPU`.
- A test image can be decoded.
- YOLO OpenVINO export succeeds.
- YOLO prediction saves an annotated image with bounding boxes.
- CPU and GPU benchmark numbers are recorded.
- NPU benchmark numbers are recorded if the model compiles on NPU.
- We can decide whether integrated Intel silicon is enough for the target FPS, latency, and power budget.

## 17. Recommended First Test Order

Use this exact order to avoid confusion:

```text
1. Confirm CPU is Core Ultra 7 255H or better.
2. Confirm Ubuntu and kernel are suitable.
3. Confirm /dev/dri exists for GPU.
4. Confirm /dev/accel/accel0 exists for NPU.
5. Install or repair GPU runtime.
6. Install or repair NPU driver.
7. Install OpenVINO in a Python virtual environment.
8. Confirm OpenVINO devices.
9. Capture or copy one test image.
10. Export small YOLO model to OpenVINO.
11. Run YOLO on CPU.
12. Run YOLO on GPU.
13. Run YOLO on NPU.
14. Run benchmark_app on CPU, GPU, NPU.
15. Run end-to-end YOLO benchmark.
16. Decide if integrated Intel silicon is enough.
```

## Source Links

- Intel Core Ultra 7 255H specifications: https://www.intel.com/content/www/us/en/products/sku/241751/intel-core-ultra-7-processor-255h-24m-cache-up-to-5-10-ghz/specifications.html
- OpenVINO 2026 Linux install options: https://docs.openvino.ai/2026/get-started/install-openvino/install-openvino-linux.html
- OpenVINO PyPI install: https://docs.openvino.ai/2026/get-started/install-openvino/install-openvino-pip.html
- OpenVINO APT install: https://docs.openvino.ai/2026/get-started/install-openvino/install-openvino-apt.html
- OpenVINO system requirements: https://docs.openvino.ai/2026/about-openvino/release-notes-openvino/system-requirements.html
- OpenVINO Intel GPU configuration: https://docs.openvino.ai/2026/get-started/install-openvino/configurations/configurations-intel-gpu.html
- OpenVINO Intel NPU configuration: https://docs.openvino.ai/2026/get-started/install-openvino/configurations/configurations-intel-npu.html
- OpenVINO NPU device guide: https://docs.openvino.ai/2026/openvino-workflow/running-inference/inference-devices-and-modes/npu-device.html
- Intel Linux NPU driver repository: https://github.com/intel/linux-npu-driver
- Intel Linux NPU driver releases: https://github.com/intel/linux-npu-driver/releases
- OpenVINO Benchmark Tool: https://docs.openvino.ai/2026/get-started/learn-openvino/openvino-samples/benchmark-tool.html
- Ultralytics OpenVINO export: https://docs.ultralytics.com/integrations/openvino
- Ultralytics export modes: https://docs.ultralytics.com/modes/export
