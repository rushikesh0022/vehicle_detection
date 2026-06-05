# DeepStream WSL Setup Guide

Last reviewed: 2026-06-03

This guide walks from a fresh Windows machine to running NVIDIA DeepStream 9.0 inside Ubuntu 24.04 on WSL 2. It is written for the Docker-based WSL path recommended by NVIDIA.

## Target Result

At the end, you should be able to open Ubuntu in WSL and run:

```bash
sudo docker run --rm --gpus all ubuntu nvidia-smi
sudo docker run -it --rm --gpus all --privileged \
  -e NVIDIA_DRIVER_CAPABILITIES=compute,utility,video,graphics \
  nvcr.io/nvidia/deepstream:9.0-triton-multiarch
```

Inside the DeepStream container, you should be able to run:

```bash
deepstream-app --version-all
```

## 0. Prerequisites

You need:

- Windows 11 with administrator access.
- Supported NVIDIA GPU. Use GeForce, NVIDIA RTX, or Quadro in WDDM mode.
- Current NVIDIA Windows GPU driver for your GPU.
- Internet access for WSL, Ubuntu packages, Docker packages, NVIDIA Container Toolkit packages, and the DeepStream NGC image.
- Enough disk space for WSL, Docker, and the DeepStream container image.

Important constraints:

- Do not install a Linux NVIDIA display driver inside WSL.
- Tesla/datacenter GPUs are not supported for this WSL flow.
- If your GPU does not appear in `nvidia-smi` inside WSL, stop and fix the Windows driver or WSL setup first.
- Prefer file output for validation because display sinks such as `nveglglessink` can show black output in Ubuntu 24.04 based DeepStream containers on WSL 2.

## 1. Install The NVIDIA Windows Driver

1. Open the NVIDIA driver download page: https://www.nvidia.com/Download/index.aspx
2. Select your GPU and Windows version.
3. Download and run the Windows driver installer.
4. Choose `Advanced` installation.
5. Choose `Perform a clean installation`.
6. Reboot Windows.

After reboot, open PowerShell and check:

```powershell
nvidia-smi
```

If Windows cannot run `nvidia-smi`, fix the driver before continuing.

## 2. Install Or Update WSL 2

Open Windows Terminal or PowerShell as Administrator.

Check the installed WSL version:

```powershell
wsl --version
```

Update WSL:

```powershell
wsl --update
```

List available distributions:

```powershell
wsl --list --online
```

Install Ubuntu 24.04:

```powershell
wsl --install Ubuntu-24.04
```

If your Windows build expects the distribution flag, use this form instead:

```powershell
wsl --install -d Ubuntu-24.04
```

If Windows asks for a reboot, reboot, then open PowerShell as Administrator again.

Confirm Ubuntu is installed:

```powershell
wsl --list --verbose
```

If Ubuntu is not using WSL 2, set it explicitly:

```powershell
wsl --set-version Ubuntu-24.04 2
```

Start Ubuntu:

```powershell
wsl -d Ubuntu-24.04
```

The first launch asks you to create a Linux username and password. This password is used for `sudo` inside Ubuntu.

## 3. Prepare Ubuntu And Validate GPU Access

Inside Ubuntu:

```bash
sudo apt update
sudo apt upgrade -y
```

Validate that WSL can see the NVIDIA GPU:

```bash
nvidia-smi
```

Expected result: an NVIDIA-SMI table showing the GPU, driver version, and CUDA version.

If `nvidia-smi` is not found, try:

```bash
/usr/lib/wsl/lib/nvidia-smi
```

If this also fails, return to Windows and update or reinstall the NVIDIA Windows driver, then run:

```powershell
wsl --shutdown
wsl -d Ubuntu-24.04
```

## 4. Install Docker Engine Inside Ubuntu

These commands follow Docker's current Ubuntu apt repository method.

Remove conflicting old Docker packages if present:

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt remove -y "$pkg"
done
```

Install repository dependencies:

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add Docker's apt repository:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources >/dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Install Docker:

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Start Docker:

```bash
sudo systemctl start docker
```

If `systemctl` is not available in your WSL environment, use:

```bash
sudo service docker start
```

Verify Docker:

```bash
sudo docker run hello-world
```

Expected result: Docker prints a confirmation message and exits.

Optional: allow your WSL user to run Docker without `sudo`.

```bash
sudo usermod -aG docker "$USER"
newgrp docker
docker run hello-world
```

Using `sudo docker ...` is fine if you prefer to keep the setup simple.

## 5. Install NVIDIA Container Toolkit

Add the NVIDIA Container Toolkit repository:

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

Install it:

```bash
sudo apt update
sudo apt install -y nvidia-container-toolkit
```

Configure Docker to use the NVIDIA runtime:

```bash
sudo nvidia-ctk runtime configure --runtime=docker
```

Restart Docker:

```bash
sudo systemctl restart docker
```

If `systemctl` is not available:

```bash
sudo service docker restart
```

Verify GPU access from Docker:

```bash
sudo docker run --rm --gpus all ubuntu nvidia-smi
```

Expected result: an NVIDIA-SMI table from inside the container.

## 6. Pull The DeepStream 9.0 Container

Pull the main DeepStream Triton container:

```bash
sudo docker pull nvcr.io/nvidia/deepstream:9.0-triton-multiarch
```

NVIDIA also lists a samples container:

```bash
sudo docker pull nvcr.io/nvidia/deepstream:9.0-samples-multiarch
```

If Docker reports an authorization error for `nvcr.io`, sign in to NVIDIA NGC, create an API key, then log in:

```bash
sudo docker login nvcr.io
```

Use:

- Username: `$oauthtoken`
- Password: your NGC API key

Then retry the `docker pull`.

## 7. Start The DeepStream Container

First create an output folder in Ubuntu:

```bash
mkdir -p "$HOME/deepstream-output"
```

Start the container with GPU access and an output mount:

```bash
sudo docker run -it --rm \
  --name deepstream-wsl \
  --net=host \
  --gpus all \
  --privileged \
  -e CUDA_CACHE_DISABLE=0 \
  -e NVIDIA_DRIVER_CAPABILITIES=compute,utility,video,graphics \
  -v "$HOME/deepstream-output:/workspace/output" \
  nvcr.io/nvidia/deepstream:9.0-triton-multiarch
```

This starts a shell inside the DeepStream container.

If you need X11 display support, install X11 helper tools in Ubuntu and add the display mounts:

```bash
sudo apt install -y x11-xserver-utils
xhost +
```

Then run:

```bash
sudo docker run -it --rm \
  --name deepstream-wsl \
  --net=host \
  --gpus all \
  --privileged \
  -e DISPLAY="$DISPLAY" \
  -e CUDA_CACHE_DISABLE=0 \
  -e NVIDIA_DRIVER_CAPABILITIES=compute,utility,video,graphics \
  --device /dev/snd \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v "$HOME/deepstream-output:/workspace/output" \
  nvcr.io/nvidia/deepstream:9.0-triton-multiarch
```

For WSL 2 validation, file output is more reliable than display output.

## 8. Verify DeepStream Inside The Container

Inside the container:

```bash
deepstream-app --version-all
```

Expected result: DeepStream prints the SDK version and related component versions.

Run a simple DeepStream/GStreamer decode and encode pipeline:

```bash
cd /opt/nvidia/deepstream/deepstream/samples/streams
gst-launch-1.0 filesrc location=sample_720p.mp4 ! qtdemux ! h264parse ! nvv4l2decoder ! nvv4l2h264enc ! h264parse ! qtmux ! filesink location=/workspace/output/filedump.mp4 -v
```

Expected result:

- The pipeline completes successfully.
- `/workspace/output/filedump.mp4` is created in the container.
- The file is also available in Ubuntu at `~/deepstream-output/filedump.mp4`.

From Windows, you can browse WSL files at:

```text
\\wsl.localhost\Ubuntu-24.04\home\<your-linux-user>\deepstream-output
```

## 9. Run The `deepstream-app` Sample With File Output

Inside the container:

```bash
cd /opt/nvidia/deepstream/deepstream/samples/configs/deepstream-app
cp source30_1080p_dec_infer-resnet_tiled_display.txt /workspace/output/source30_wsl_file.txt
```

Edit the copied config:

```bash
vi /workspace/output/source30_wsl_file.txt
```

Make these changes:

- In `[sink0]`, set `enable=0`.
- In `[sink1]`, set `enable=1`.
- Confirm the file sink writes to a usable path, for example `output-file=/workspace/output/out.mp4` if that key exists in the sink section.

Run the app:

```bash
deepstream-app -c /workspace/output/source30_wsl_file.txt
```

Expected result:

- DeepStream runs inference on the sample streams.
- An output MP4 such as `out.mp4` is written to `/workspace/output`.
- The output appears in Ubuntu at `~/deepstream-output`.

If the app tries to display a window and you see black output or display errors, keep `[sink0]` disabled and use a file sink.

## 10. YOLO COCO Model Path For DeepStream Optional

Use this section after the basic DeepStream checks in sections 8 and 9 pass.

DeepStream can run YOLO models, but the model is not used in the same direct way as a Python `yolo predict ...` command. DeepStream normally runs inference through TensorRT by using `Gst-nvinfer` or `Gst-nvinferserver`, so a YOLO setup needs:

- A TensorRT-compatible model artifact, usually ONNX or a generated TensorRT engine.
- A DeepStream `nvinfer` config file.
- A YOLO parser or postprocessing library that understands the model output.
- A label file, such as the 80-class COCO label list.
- A normal input video, image, webcam, or RTSP stream.

Important: a pre-segmented video is not required. The input video is a normal video. If the model is a YOLO segmentation model, the model generates masks during inference and DeepStream overlays or exports the result depending on the pipeline configuration.

### Choose The Correct YOLO Model

For COCO object detection, use a COCO-pretrained detection checkpoint, for example:

```text
yolo26n.pt
yolo11n.pt
yolov8n.pt
```

For COCO instance segmentation, use a COCO-pretrained segmentation checkpoint, for example:

```text
yolo26n-seg.pt
yolo11n-seg.pt
yolov8n-seg.pt
```

The `-seg` part matters. A plain detection model produces boxes and classes only. It will not produce segmentation masks.

If you already trained your own model, use your trained `best.pt` and confirm whether it is a detection model or a segmentation model before exporting it.

### Export YOLO To ONNX Before DeepStream

You can export from Windows, from WSL Ubuntu, or from another Linux machine. The output file should then be copied into the folder that you mount into the DeepStream container.

Inside Ubuntu WSL, one simple export workflow is:

```bash
sudo apt update
sudo apt install -y python3-venv python3-pip

python3 -m venv "$HOME/yolo-export-venv"
source "$HOME/yolo-export-venv/bin/activate"
python -m pip install --upgrade pip
python -m pip install ultralytics onnx

mkdir -p "$HOME/deepstream-work/models/yolo-coco"
yolo export model=yolo26n-seg.pt format=onnx imgsz=640 simplify=True opset=17
cp yolo26n-seg.onnx "$HOME/deepstream-work/models/yolo-coco/"
```

For a detection-only model, replace `yolo26n-seg.pt` with the detection model name, such as `yolo26n.pt`. For your own model, replace it with the path to your `best.pt`.

### Prepare The DeepStream YOLO Files

Inside the WSL project folder, keep the model, labels, parser library, and configs together:

```text
~/deepstream-work/
  models/
    yolo-coco/
      yolo26n-seg.onnx
      labels.txt
  configs/
    yolo-coco/
      config_infer_primary_yolo.txt
      deepstream_app_config_yolo_coco.txt
  yolo-parser/
    libnvdsinfer_custom_impl_yolo.so
```

The parser library name depends on the YOLO implementation you use. NVIDIA documents that DeepStream custom models can use a custom parser library through `custom-lib-path`, and `nvinfer` loads the parser function named in the config.

The `labels.txt` file must match the class order used by the model. For a standard COCO YOLO model, use the 80 COCO class names in the same order expected by that model. For a custom model, use your training dataset class order.

For YOLO detection, the parser must produce bounding boxes and class IDs.

For YOLO instance segmentation, the parser must produce bounding boxes, class IDs, and masks. In DeepStream `nvinfer`, instance segmentation uses `network-type=3`, `output-instance-mask=1`, and a `parse-bbox-instance-mask-func-name` implementation.

### Example `nvinfer` Config Shape

Create:

```bash
vi "$HOME/deepstream-work/configs/yolo-coco/config_infer_primary_yolo.txt"
```

Use this as a checklist-style starting point, then fill in the exact parser function names required by your YOLO parser library:

```ini
[property]
gpu-id=0
onnx-file=/workspace/work/models/yolo-coco/yolo26n-seg.onnx
model-engine-file=/workspace/work/models/yolo-coco/yolo26n-seg_b1_gpu0_fp16.engine
labelfile-path=/workspace/work/models/yolo-coco/labels.txt
batch-size=1
network-mode=2
num-detected-classes=80
gie-unique-id=1
process-mode=1
model-color-format=0
net-scale-factor=0.0039215697906911373
maintain-aspect-ratio=1
symmetric-padding=1

# Use this for YOLO detection models.
# network-type=0
# custom-lib-path=/workspace/work/yolo-parser/libnvdsinfer_custom_impl_yolo.so
# parse-bbox-func-name=<your_yolo_detection_parser_function>
# cluster-mode=2

# Use this for YOLO instance segmentation models.
network-type=3
output-instance-mask=1
custom-lib-path=/workspace/work/yolo-parser/libnvdsinfer_custom_impl_yolo.so
parse-bbox-instance-mask-func-name=<your_yolo_segmentation_parser_function>
```

Do not leave the placeholder parser function names in the final file. Replace them with the exact function exported by the parser library you build or reuse.

### Run YOLO Through `deepstream-app`

Start the container with the working folder mounted:

```bash
sudo docker run -it --rm \
  --name deepstream-wsl \
  --net=host \
  --gpus all \
  --privileged \
  -e CUDA_CACHE_DISABLE=0 \
  -e NVIDIA_DRIVER_CAPABILITIES=compute,utility,video,graphics \
  -v "$HOME/deepstream-work:/workspace/work" \
  -v "$HOME/deepstream-output:/workspace/output" \
  nvcr.io/nvidia/deepstream:9.0-triton-multiarch
```

Inside the container, copy a known-working DeepStream app config and point its `[primary-gie]` section to the YOLO config:

```bash
cd /opt/nvidia/deepstream/deepstream/samples/configs/deepstream-app
cp source30_1080p_dec_infer-resnet_tiled_display.txt /workspace/work/configs/yolo-coco/deepstream_app_config_yolo_coco.txt
vi /workspace/work/configs/yolo-coco/deepstream_app_config_yolo_coco.txt
```

Edit the copied app config:

- In `[primary-gie]`, set `config-file=/workspace/work/configs/yolo-coco/config_infer_primary_yolo.txt`.
- Use one normal input video first, such as a DeepStream sample stream or your own MP4.
- Disable display sinks for WSL validation.
- Enable a file sink and write the result to `/workspace/output/yolo_coco_out.mp4`.

Run:

```bash
deepstream-app -c /workspace/work/configs/yolo-coco/deepstream_app_config_yolo_coco.txt
```

Expected result:

- The first run may take several minutes while TensorRT builds the engine file.
- A TensorRT `.engine` file appears beside the ONNX model if engine generation succeeds.
- The output MP4 is written under `/workspace/output` and appears in Ubuntu at `~/deepstream-output`.
- Detection models should show boxes and labels. Segmentation models should show masks only if the parser and OSD path support instance masks.

### Using Existing YOLO DeepStream Helpers

There are helper repositories that provide YOLO parsers and sample configs for DeepStream. They can save time, but version compatibility matters.

- NVIDIA's DeepStream custom model documentation explains the official mechanism: configure `nvinfer`, provide ONNX or engine files, and use custom parser functions when needed.
- The community `DeepStream-Yolo` project supports many modern YOLO families, including YOLOv8, YOLO11, and YOLO26 style models, but its README currently lists support up to DeepStream 8.0. If you use DeepStream 9.0, verify compatibility before relying on it.
- Many YOLO DeepStream examples are detection-only. For segmentation, confirm that the parser supports instance masks, not just bounding boxes.

Recommended validation order:

1. Run the built-in DeepStream sample from section 9.
2. Export the COCO YOLO model to ONNX.
3. Build or select a parser that matches your YOLO family and task.
4. Run one normal MP4 through DeepStream and write output to a file.
5. Only after the COCO model works, replace it with your original model.

## 11. Daily Usage Pattern

Start Ubuntu from PowerShell:

```powershell
wsl -d Ubuntu-24.04
```

Inside Ubuntu, start Docker if needed:

```bash
sudo service docker start
```

Run the DeepStream container with your working folder mounted:

```bash
mkdir -p "$HOME/deepstream-work" "$HOME/deepstream-output"
sudo docker run -it --rm \
  --name deepstream-wsl \
  --net=host \
  --gpus all \
  --privileged \
  -e CUDA_CACHE_DISABLE=0 \
  -e NVIDIA_DRIVER_CAPABILITIES=compute,utility,video,graphics \
  -v "$HOME/deepstream-work:/workspace/work" \
  -v "$HOME/deepstream-output:/workspace/output" \
  nvcr.io/nvidia/deepstream:9.0-triton-multiarch
```

Recommended file locations:

- Keep active Linux projects under the WSL filesystem, such as `/home/<user>/deepstream-work`.
- Avoid heavy Docker or build workloads directly under `/mnt/c/...` because Windows-mounted paths are usually slower.
- Access WSL files from Windows through `\\wsl.localhost\Ubuntu-24.04\home\<user>`.

To stop:

```bash
exit
```

To fully restart WSL from PowerShell:

```powershell
wsl --shutdown
wsl -d Ubuntu-24.04
```

## Troubleshooting

### `nvidia-smi` fails inside Ubuntu

Fix the Windows driver first. Do not install a Linux NVIDIA display driver inside WSL. After updating the Windows driver, run:

```powershell
wsl --shutdown
wsl -d Ubuntu-24.04
```

Then test `nvidia-smi` again inside Ubuntu.

### Docker says it cannot select a GPU device driver

Common error:

```text
could not select device driver "" with capabilities: [[gpu]]
```

Run:

```bash
sudo apt install --reinstall -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo service docker restart
```

Then from PowerShell:

```powershell
wsl --shutdown
wsl -d Ubuntu-24.04
```

Retry:

```bash
sudo docker run --rm --gpus all ubuntu nvidia-smi
```

### Docker service is not running

Try:

```bash
sudo service docker start
```

If your WSL has systemd enabled:

```bash
sudo systemctl start docker
```

### DeepStream display is black or `nveglglessink` fails

This is a known WSL 2 issue for Ubuntu 24.04 based DeepStream containers. Use `filesink` and generate MP4 output instead of validating with display sinks.

### WSL install fails with a Windows Update service error

Open `services.msc` as Administrator, find `Windows Update`, set startup type to `Manual` or `Automatic`, start the service, then retry the WSL install command.

### First DeepStream run prints warnings

NVIDIA documents some first-run warnings from missing optional OSS plugin libraries inside the container. If `deepstream-app --version-all` works and the sample pipeline generates the output file, these warnings can usually be ignored.

### YOLO model runs but no masks appear

Confirm the model is actually a segmentation model, such as `yolo26n-seg.pt`, `yolo11n-seg.pt`, `yolov8n-seg.pt`, or a segmentation-trained `best.pt`. A detection-only model cannot output masks.

Also confirm the DeepStream parser supports instance segmentation. For `nvinfer`, instance segmentation requires `network-type=3`, `output-instance-mask=1`, and a valid `parse-bbox-instance-mask-func-name`.

### YOLO parser fails to load

Check that the parser `.so` file exists inside the container, that `custom-lib-path` points to the container path, and that the parser function name in the config exactly matches the exported function in the library.

For DeepStream 9.0, also verify that any third-party YOLO helper repo you use supports the DeepStream, CUDA, and TensorRT versions in your container.

### Pulling from `nvcr.io` fails

Sign in to NVIDIA NGC, create an API key, and run:

```bash
sudo docker login nvcr.io
```

Then pull again:

```bash
sudo docker pull nvcr.io/nvidia/deepstream:9.0-triton-multiarch
```

## Setup Checklist

- [ ] NVIDIA Windows driver installed and Windows `nvidia-smi` works.
- [ ] WSL 2 installed and updated.
- [ ] Ubuntu 24.04 installed.
- [ ] Ubuntu `nvidia-smi` works.
- [ ] Docker Engine installed in Ubuntu.
- [ ] `sudo docker run hello-world` works.
- [ ] NVIDIA Container Toolkit installed.
- [ ] `sudo docker run --rm --gpus all ubuntu nvidia-smi` works.
- [ ] DeepStream image pulled.
- [ ] `deepstream-app --version-all` works inside the container.
- [ ] Sample pipeline writes `filedump.mp4`.
- [ ] `deepstream-app` sample writes an output MP4 through a file sink.
- [ ] YOLO COCO model is exported to ONNX if YOLO validation is required.
- [ ] YOLO parser and `nvinfer` config are selected for detection or instance segmentation.
- [ ] YOLO DeepStream run writes an output MP4 before replacing the COCO model with the original model.

## Source Links

- NVIDIA DeepStream on WSL: https://docs.nvidia.com/metropolis/deepstream/dev-guide/text/DS_on_WSL2.html
- NVIDIA FAQ for DeepStream on WSL: https://docs.nvidia.com/metropolis/deepstream/9.0/text/DS_WSL2_FAQ.html
- NVIDIA DeepStream Docker containers: https://docs.nvidia.com/metropolis/deepstream/dev-guide/text/DS_docker_containers.html
- NVIDIA DeepStream installation guide: https://docs.nvidia.com/metropolis/deepstream/dev-guide/text/DS_Installation.html
- NVIDIA DeepStream custom models: https://docs.nvidia.com/metropolis/deepstream/9.0/text/DS_using_custom_model.html
- NVIDIA DeepStream `Gst-nvinfer`: https://docs.nvidia.com/metropolis/deepstream/9.0/text/DS_plugin_gst-nvinfer.html
- NVIDIA DeepStream Inference Builder: https://docs.nvidia.com/metropolis/deepstream/9.0/text/DS_Inference_Builder.html
- Community DeepStream-Yolo helper project: https://github.com/marcoslucianops/DeepStream-Yolo
- NVIDIA CUDA on WSL user guide: https://docs.nvidia.com/cuda/wsl-user-guide/index.html
- NVIDIA Container Toolkit install guide: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
- Microsoft WSL install guide: https://learn.microsoft.com/en-us/windows/wsl/install
- Microsoft WSL basic commands: https://learn.microsoft.com/en-us/windows/wsl/basic-commands
- Docker Engine on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Docker GPU access: https://docs.docker.com/engine/containers/gpu/
