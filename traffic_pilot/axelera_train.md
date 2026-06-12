# Axelera ALPR — How to Run the Cascade YAML, and How to Train an Indian OCR

Two parts:
- **Part A** — run the existing 3-stage cascade YAML now (vehicle→plate→OCR on the AIPU).
  Works today; OCR text is the Chinese placeholder.
- **Part B** — train an **Indian** LPRNet on your own dataset and compile it to the AIPU so
  the OCR text is correct. This is the only way to get real Indian plate text on Axelera.

> Verified first: there is **no pre-made OCR model for the AIPU** anywhere on this machine.
> Every `.axnet` on the box is a detector (YOLO/vehicle/plate) or an ImageNet classifier.
> `models/axelera/license_plate` is plate **detection** (output `[1,300,6]`), not OCR. The
> only OCR that exists is `cct.onnx`, which is OpenVINO-only. So Part B is required for real
> Indian text on Axelera.

---

# Part A — Run the cascade YAML (works today, Chinese OCR text)

The cascade is `config/cascade/vehicle-plate-lpr-cascade.yaml`. It runs through the **Voyager
Python tools** (`deploy.py` to compile, `inference.py` to run). The C++ daemon cannot run
this cascade — only the Python path can.

### 1. Activate the SDK
```bash
cd /home/admin1/voyager-sdk
source venv/bin/activate
```

### 2. Compile (deploy) — one-time, ~10–30 min, downloads the LPRNet weights
`deploy.py` converts each of the 3 models (vehicle YOLO, plate YOLO, LPRNet) into AIPU
`.axnet` binaries and writes a build folder. It downloads `Final_LPRNet_model.pth` from
Axelera's server on first run.
```bash
./deploy.py /home/admin1/Desktop/traffic/traffic-pilot/config/cascade/vehicle-plate-lpr-cascade.yaml
```
Watch the per-model progress. When it finishes, a build root exists (e.g. under
`/home/admin1/voyager-sdk/build/alpr-vehicle-plate-lpr/`).

### 3. Run (inference) on your video
```bash
./inference.py \
  /home/admin1/Desktop/traffic/traffic-pilot/config/cascade/vehicle-plate-lpr-cascade.yaml \
  <your-video-or-rtsp>
# e.g. a file:  media/your_clip.mp4      or an RTSP url
```
You'll see vehicle boxes, plate boxes, and OCR text overlaid. **Expect the OCR text to be
wrong** on Indian plates (Chinese charset) — this run is for proving the 3-stage AIPU
cascade works and for measuring throughput, not for reading plates.

### What Part A gives you
- ✅ Confirms the full all-Axelera cascade runs end-to-end on the AIPU.
- ✅ Throughput numbers (fps, OCR reads/s) for the all-Axelera benchmark vs OpenVINO.
- ❌ Not correct Indian plate text.

---

# Part B — Train an Indian LPRNet and compile it to the AIPU

Goal: replace the Chinese LPRNet with one trained on Indian plates, so the AIPU OCR text is
correct. Then the all-Axelera mode genuinely matches OpenVINO.

## The dataset you have

`datasets/plate_ocr/` — **100** pairs: `<uuid>.jpg` (a cropped plate) + `<uuid>.txt` (the
plate string, e.g. `TS10EE5625`). This is real Indian data and is the right *shape* for OCR
training (image → string).

**⚠ Honest blocker: 100 images is far too few to train an OCR from scratch.** Real LPRNet
training uses tens of thousands of images. Before training, do ONE of:
1. **Find the full set** — `cct.onnx` was trained on Indian data; locate that larger
   training set (this 100 is likely just a calibration/eval slice). *Start here.*
2. **Generate synthetic Indian plates** — render `LL DD LL DDDD` strings on plate templates
   with augmentation (blur, perspective, lighting) to reach 20k–100k images.
3. **Fine-tune, don't train from scratch** — start from the Chinese LPRNet weights, swap the
   final classifier to the Indian charset, and fine-tune on whatever real data you have.
   Needs the least data but still more than 100 for good accuracy.

## Step 1 — Define the Indian charset (37 classes, Latin)

Indian plates use `0-9` and `A-Z` only — no Chinese glyphs. Match `cct.onnx`'s 37 classes
(36 chars + 1 CTC blank). Create:

`config/cascade/labels/lprnet-char-india.yaml`
```yaml
names:
  0: '0'
  1: '1'
  # ... 2-9 ...
  10: A
  11: B
  # ... C-Z ...
  35: Z
  36: ''        # CTC blank
```
(No province glyphs. `lpr_max_len` should be ~10 for Indian format, not 8.)

## Step 2 — Put the dataset in LPRNet training layout

The Voyager LPRNet data adapter is
`/home/admin1/voyager-sdk/ax_datasets/license_plate_recog.py` (`LPRNetDataAdapter`,
`data_dir_name: license_plate_recog`). Arrange your images + label strings the way that
adapter expects (image filename or sidecar holds the plate string). Your `.jpg`+`.txt`
pairs convert directly — one script to copy them into the adapter's expected folder/label
format.

## Step 3 — Train the LPRNet (PyTorch, off the AIPU)

Training happens in normal PyTorch, not on the AIPU. Use the model definition at
`/home/admin1/voyager-sdk/ax_models/torch/LPRNet.py` (or a standard LPRNet repo), with:
- `num_classes = 37` (Indian charset above)
- `lpr_max_len ≈ 10`
- CTC loss, your Indian dataset (real + synthetic), standard augmentation.

Output: an Indian `Final_LPRNet_india.pth`. Validate character/plate accuracy against a
held-out Indian set **before** compiling — this is the gate.

## Step 4 — Make an Indian cascade YAML

Copy the existing cascade and point the OCR stage at the Indian model + charset:

`config/cascade/vehicle-plate-lpr-india.yaml` (based on `vehicle-plate-lpr-cascade.yaml`):
```yaml
  lprnet:
    class: LPRNetTorchModel
    class_path: $AXELERA_FRAMEWORK/ax_models/torch/LPRNet.py
    weight_path: <path>/Final_LPRNet_india.pth     # your trained Indian weights
    num_classes: 37                                # was 68
    dataset: LPRNetIndiaDataset                    # points at lprnet-char-india.yaml
    extra_kwargs:
      LPRNet:
        lpr_max_len: 10                            # Indian length, was 8
```
Keep the vehicle and plate stages unchanged (they already work).

## Step 5 — Compile and run

```bash
cd /home/admin1/voyager-sdk && source venv/bin/activate
./deploy.py /home/admin1/Desktop/traffic/traffic-pilot/config/cascade/vehicle-plate-lpr-india.yaml
./inference.py /home/admin1/Desktop/traffic/traffic-pilot/config/cascade/vehicle-plate-lpr-india.yaml <video>
```
Now the AIPU OCR reads Indian plates. This is the true "complete Axelera = OpenVINO" mode.

## Realistic effort & expectation

| Step | Effort | Risk |
|---|---|---|
| Find full training set | hours–days | may not exist → need synthetic |
| Synthetic data (if needed) | 1–3 days | quality must match real plates |
| Train LPRNet | hours (GPU) per run, several runs | accuracy depends on data |
| New YAML + compile | ~1 hour + deploy time | low |
| Validate accuracy | ongoing | **the real gate** |

Expect Indian-LPRNet OCR accuracy to be **lower than your `cct.onnx`** at first — LPRNet is
a smaller/older architecture than CCT. The tradeoff is: LPRNet runs on the AIPU, CCT does
not. Decide if the accuracy is good enough for the all-Axelera benchmark, or keep CCT on
OpenVINO for production-grade text.

---

## Quick decision guide

- **Just want the all-Axelera speed number now?** → Part A.
- **Want real Indian text on the AIPU?** → Part B, and **first** go find the full OCR
  training set (`cct.onnx`'s data). 100 images alone is not enough.
- **Want best accuracy regardless of "pure Axelera"?** → keep OCR on OpenVINO `cct.onnx`
  (today's hybrid), vehicle+plate on Axelera.
