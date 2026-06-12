# OpenVINO-only Pipeline — Improvement Plan

Scope: the **OpenVINO-only** cascade (`./start_openvino.sh`) — vehicle on the Intel
**iGPU**, plate-detection + OCR on the Intel **NPU**. No Axelera AIPU.

This is the plan for breaking the current throughput wall: what is true today, what to
do, what each change costs in accuracy, and the FPS we expect back.

> Numbers tagged **(measured)** come from live 15-camera runs. Numbers tagged
> **(estimate)** are reasoned guesses from chip utilization / quantization behavior and
> **must be confirmed with a benchmark capture** before being trusted. Accuracy figures
> in particular MUST be validated on our own plate dataset — do not ship an INT8 model on
> a quoted number.

---

## 1. Where we are today (the baseline)

| Stage    | Device | Model (on disk)                 | Runs at |
|----------|--------|---------------------------------|---------|
| Vehicle  | iGPU   | `vehicle_yolo26n_640.onnx` FP32 | FP16    |
| Plate    | NPU    | `plate_yolo26n_224_dyn.onnx` FP32 | FP16  |
| OCR      | NPU    | `cct.onnx` FP32                 | FP16    |

**Measured (15 cams):**
- Per-camera throughput: **~7–8 fps/cam**
- `OV-callback`: ~440–530 frames / 5s on the iGPU vehicle stage
- **iGPU (CCS / compute engine): ~90–98%** — saturated
- **NPU: ~90–100%** — saturated
- CPU ~40%, RAM ~26% — **idle, not the bottleneck**

**Both accelerators are the wall at the same time.** That is the single most important
fact for this plan: there is no idle chip to offload to. Work has to get *cheaper*, not
*moved*.

---

## 2. Two facts that correct common misconceptions

**(a) FP32-on-disk does NOT cost extra runtime compute.**
The Intel NPU (AI Boost / 3720) has no FP32 execution path — it runs everything in
**FP16**. The FP32→FP16 down-conversion happens **once, at compile time** when the blob
is built, not per-frame. So the models being FP32 files is *not* where the cost is; at
runtime they are already pure FP16. The only way to make the NPU itself faster is
**INT8** (different math units), not anything about the FP32 source file.

**(b) Moving OCR to the iGPU does NOT help in OpenVINO-only mode.**
That trick works in *hybrid* mode because there the iGPU is idle (vehicle is on the
Axelera chip). In OpenVINO-only the iGPU is **already the co-bottleneck** doing vehicle.
Putting OCR on it just pins an already-full chip and wastes the freed NPU headroom.
- **Estimate:** roughly flat to slightly worse (−5% to +5%) — a wash. **Do not do this
  in OpenVINO-only mode.**

---

## 3. What to do (ordered by ROI)

### Step 1 — Skip-frame OCR  ★ do this first (free, no accuracy loss)
A plate does not change between consecutive frames of the same vehicle track. Today every
delivered frame of a track is OCR'd. Instead: OCR a track every Nth frame (or until the
OcrStabilizer confirms the plate, then stop OCR'ing that track).

- **Effort:** code change in the daemon's per-track plate path (gate OCR by track-age /
  confirmed-state). No model changes, no retraining.
- **Accuracy impact:** **none** to the *published* plate — the stabilizer already needs
  multiple votes; we just stop re-reading a plate we've already locked.
- **FPS (estimate):** cuts OCR load **3–5×**. Because OCR shares the NPU with
  plate-detect, this frees meaningful NPU budget. Expect **per-cam ~7–8 → ~9–11 fps**.
- **Risk:** low. Tune N so a fast-moving plate still gets enough reads to confirm.

### Step 2 — INT8 quantize the plate detector (NPU)
The NPU 3720 has dedicated INT8 MAC units → ~2× the FP16 throughput.

- **Effort:** NNCF post-training quantization (PTQ) on `plate_yolo26n_224_dyn.onnx` with a
  calibration set of real plate crops. Produces an INT8 model; recompile the NPU blob.
- **Accuracy impact (estimate, validate!):** YOLO detection is robust to INT8 —
  typically **1–3% mAP** drop. The stabilizer absorbs most per-frame noise.
- **FPS (estimate):** ~2× the plate-detect path on the NPU → combined with Step 1,
  **per-cam toward ~11–14 fps**.
- **Risk:** low-medium. Must measure mAP on held-out plates before trusting.

### Step 3 — INT8 quantize the vehicle YOLO (iGPU)  ★ the forgotten wall
In OpenVINO-only mode the **iGPU is just as much the bottleneck as the NPU**. INT8 on the
iGPU vehicle model lifts the *other* wall — most people only quantize the NPU and leave
the iGPU pinned.

- **Effort:** NNCF PTQ on `vehicle_yolo26n_640.onnx` with a calibration set of road
  frames. Arc iGPU has strong INT8 (DP4a/XMX) throughput.
- **Accuracy impact (estimate, validate!):** **1–3% mAP** on vehicle detection. Vehicles
  are large, easy targets — low risk.
- **FPS (estimate):** iGPU vehicle throughput **~1.5–2×**. This is what lets the *whole*
  cascade scale, since vehicle gates everything downstream.
- **Risk:** low-medium.

### Step 4 — INT8 OCR, or keep OCR at FP16  ⚠ most sensitive — do last
OCR is where INT8 hurts most: one wrong character fails the whole plate.

- **Accuracy impact (estimate, validate!):** **1–5% per-character** drop. The stabilizer
  helps, but this is the risky one.
- **Recommendation:** keep OCR at **FP16** until Steps 1–3 are measured. Only INT8 the
  OCR if you still need NPU headroom AND validation on real plates shows an acceptable
  character-accuracy drop. **Quantize selectively** — INT8 the detectors, FP16 the OCR.

### Step 5 — Smaller vehicle input (optional, if iGPU still the wall)
Drop vehicle input 640→512.
- **Accuracy impact:** small recall loss on *distant / small* vehicles only.
- **FPS (estimate):** notable iGPU saving (compute scales ~with pixels²).
- **Risk:** medium — verify small-vehicle recall on our footage.

---

## 4. What we will NOT do (and why)

| Idea | Verdict in OpenVINO-only |
|------|--------------------------|
| Move OCR NPU→iGPU | ❌ iGPU already saturated — wash or worse (a hybrid-only trick) |
| Convert FP32 files → FP16 files | ❌ no runtime gain — NPU already executes FP16 |
| Throw more CPU/RAM at it | ❌ CPU ~40%, RAM ~26% — never the bottleneck |

---

## 5. Expected end state

| Configuration | Per-cam FPS (15 cams) | Notes |
|---------------|----------------------|-------|
| **Today** (FP16, OCR every frame) | **~7–8** *(measured)* | iGPU + NPU both ~90–100% |
| + Step 1 (skip-frame OCR) | ~9–11 *(estimate)* | free, no accuracy loss |
| + Step 2 (INT8 plate) | ~11–14 *(estimate)* | ~1–3% mAP plate |
| + Step 3 (INT8 vehicle) | **~13–16** *(estimate)* | ~1–3% mAP vehicle; lifts the iGPU wall |
| + Step 4 (INT8 OCR) | headroom for **~25–30 cams** | only if OCR accuracy validates |

Alternative read of the same gains: instead of higher fps at 15 cams, hold ~7–8 fps and
roughly **double the camera count** (~25–30 cams) on the same hardware.

---

## 6. How to validate every step (do not skip)

1. **FPS:** capture a benchmark window before/after each change — the `/bench` page or
   `tools/bench/cascade_bench.py --window 8 --note "<step>"`. Both runs land in
   `bench/cascade_log.jsonl`, tagged by backend, for side-by-side comparison.
2. **Utilization:** while the window runs, watch `intel_gpu_top` (**CCS** column = iGPU
   compute), the NPU sysfs counter
   (`/sys/devices/pci*/*/npu_busy_time_us`), and `htop`. Confirm the bottleneck actually
   moved.
3. **Accuracy (the gate for any INT8 step):** run the INT8 model against a held-out set of
   **real plates** and compare detection mAP / character accuracy to the FP16 baseline.
   **No INT8 model ships on an estimated number.**

---

## 7. Recommended sequence

1. **Step 1 (skip-frame OCR)** — free, both modes, no accuracy cost. Measure.
2. **Step 3 (INT8 vehicle / iGPU)** — lifts the iGPU wall that OpenVINO-only mode is
   uniquely bound by. Measure mAP.
3. **Step 2 (INT8 plate / NPU)** — measure mAP.
4. **Step 4 (INT8 OCR)** — only if more headroom is needed and character accuracy
   validates. Otherwise keep OCR FP16.
5. **Step 5 (512 vehicle input)** — only if the iGPU is still the wall after INT8.

All device choices are already env-driven (`VEHICLE_DEVICE` / `PLATE_DEVICE` /
`OCR_DEVICE` in `run_openvino_daemon.sh`), so device experiments need only a flag flip and
a restart — no rebuild.
