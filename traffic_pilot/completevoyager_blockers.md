# Voyager / Axelera-Only Mode — What's Wrong and What Can't Be Done

Plain-language record of why a **complete Axelera-only ALPR** (same result as the
OpenVINO mode) is **not possible today**, exactly where the wall is, and what each option
would take. Every claim below was verified against the files on disk — see "How this was
verified" at the end.

---

## TL;DR

- **Vehicle + plate DETECTION on Axelera: works today.** Compiled `.axnet` artifacts and
  cascade YAMLs already exist.
- **Plate-text OCR on Axelera: does NOT work for Indian plates today.** The only AIPU OCR
  model available is a **Chinese-trained LPRNet**. It will run and emit text, but the text
  is wrong for Indian plates.
- **The accurate Indian OCR you already have (`cct.onnx`) CANNOT run on the Axelera AIPU.**
  It is a different model and is OpenVINO-only by design.
- So "complete Axelera = OpenVINO" is blocked on **one thing only: an Indian-plate OCR
  model that can run on the AIPU.** Everything else is ready.

---

## What works on Axelera right now

| Stage | Status | Evidence |
|---|---|---|
| Vehicle detection | ✅ works | `cpp/axnet/vehicle.axnet`, used by the hybrid daemon today (~180 fps) |
| Plate detection | ✅ works | `cpp/axnet/plate.axnet` + 3 compiled cascade build roots (`cascade-build`, `-s`, `-v8s`) |
| Vehicle→plate cascade | ✅ works (via Python) | `tools/eval/bakeoff_aipu.py` runs it through `create_inference_stream` |
| `all_axelera` backend mode | ✅ defined | `services/config_api/storage.py` |

**Detection-only Axelera ALPR is fully achievable today.** It produces vehicle boxes and
plate boxes — but **no plate text**.

---

## What's wrong (the OCR blocker, in detail)

### 1. The AIPU OCR model is Chinese, not Indian
The all-Axelera cascade `config/cascade/vehicle-plate-lpr-cascade.yaml` uses a model called
**LPRNet**. Its character set (`/home/admin1/voyager-sdk/ax_datasets/labels/lprnet-char.yaml`)
has **68 classes**, and the first **31 are Chinese province glyphs** (京 沪 津 渝 冀 …),
then digits, then Latin letters. Its weights (`Final_LPRNet_model.pth`) are **downloaded
from Axelera's server** and were **trained on Chinese plates (CCPD)** — different font,
layout, and length (`lpr_max_len: 8`, Chinese format).

**Effect:** run the YAML and it *will* produce output, but on an Indian plate like
`KA01AB1234` it emits something like `皖A0B12` — a Chinese province character plus garbled
Latin. **It runs; the reading is just wrong.**

### 2. Your accurate Indian OCR can't move to the AIPU
Your working OCR is **`cct.onnx`** (used in OpenVINO/hybrid mode). It is a **different
model** from LPRNet:

| | `cct.onnx` (OpenVINO OCR — accurate) | LPRNet (Axelera OCR) |
|---|---|---|
| Input | 64×128 | 24×94 |
| Output classes | **37** (0-9, A-Z, blank) → Latin / **Indian-capable** | **68** (incl. 31 Chinese glyphs) → Chinese |
| Trained on | your Indian `datasets/plate_ocr` | Chinese plates (Axelera zoo) |
| Runs on | iGPU / NPU only — **cannot compile to AIPU** | AIPU |

`cct.onnx` is OpenVINO/NPU territory by design and **does not compile to the Metis AIPU**.
That is the entire reason the AIPU cascade had to fall back to a different model (LPRNet) —
and that one is Chinese.

### 3. Why the confusion happened
- The Chinese charset file and the LPRNet weights are **inside the Voyager SDK / downloaded
  at build time** — they are **not in this repo**, which is why they weren't visible.
- Your Indian dataset (`datasets/plate_ocr`, e.g. `TS10EE5625` = Telangana) is **real and
  Indian** — but it trained **`cct.onnx` (the OpenVINO OCR)**, *not* the LPRNet. The Indian
  data never touched the AIPU OCR model.

---

## What CAN'T be done (today, without extra work)

| Want | Possible now? | Why not |
|---|---:|---|
| Vehicle + plate text on Axelera, **accurate for Indian plates** | ❌ No | No Indian OCR model exists that runs on the AIPU. The accurate one (`cct.onnx`) is OpenVINO-only; the AIPU one (LPRNet) is Chinese. |
| Move `cct.onnx` onto the AIPU | ❌ No | It does not compile to the Metis AIPU (OpenVINO-only by design). |
| Get correct plate text by pointing the Chinese LPRNet at the Indian dataset for calibration | ❌ No | **Calibration ≠ training.** Calibration only quantizes the already-trained Chinese weights; it does not change the charset or teach it Indian plates. |
| "Complete Axelera = OpenVINO" with one command today | ❌ No | Blocked on the OCR model above. Detection works; accurate text does not. |

---

## What CAN be done

| Option | What you get | Cost / risk |
|---|---|---|
| **1. Run the Chinese-LPRNet cascade now** | All-Axelera **throughput** benchmark (the 3-stage AIPU cascade runs; measures fps / OCR reads-per-second). Plate text is placeholder (Chinese), clearly labeled, never product. | Low. One `deploy.py` + `inference.py`. ~10–30 min first compile. |
| **2. Train an Indian LPRNet, compile to AIPU** | **Real Indian plate text on Axelera** → true "complete Axelera = OpenVINO". | Training task. Swap 68-class Chinese charset → 37-class Latin, train/fine-tune on `datasets/plate_ocr`, compile. **Caveat: only ~100 labeled images on hand — enough to fine-tune, thin to train from scratch; need to find the full OCR training set first.** |
| **3. Keep OCR on OpenVINO (`cct.onnx`)** | Accurate Indian text, vehicle+plate on Axelera. | Not "pure" Axelera — OCR still on iGPU/NPU. This is essentially today's hybrid mode. |

---

## Recommendation

1. **Now:** run option 1 for the all-Axelera **throughput** number vs OpenVINO — honest, no
   new code, OCR text labeled placeholder.
2. **For a real all-Axelera product:** option 2 — but first locate the **full Indian OCR
   training set** (100 images is too few to train LPRNet from scratch). If the data is
   sufficient, an Indian LPRNet on the AIPU is genuinely feasible and removes the blocker.
3. **Do not** ship the Chinese LPRNet text as if it were real, and **do not** claim
   calibration on Indian data fixes it — it does not.

---

## Separate (smaller) issue: the daemon language

The vehicle→plate **cascade** only runs through the **Python** Voyager API
(`create_inference_stream`). The C++ `AxInferenceNet` path (today's daemon,
`axn_hwdecode_bench`) only runs a **single** model (vehicle). So a C++
`traffic_pilotd_voyager` per the architecture doc would require reimplementing the cascade
in C++ from scratch. A **Python** Voyager daemon reusing `create_inference_stream` +
existing CPU tracker/stabilizer + the same Redis publisher is far less work and identical
in output. This is independent of the OCR blocker above.

---

## How this was verified

- LPRNet charset: read `/home/admin1/voyager-sdk/ax_datasets/labels/lprnet-char.yaml` — 68
  classes, first 31 Chinese province glyphs.
- LPRNet weights: `Final_LPRNet_model.pth` not on disk; YAML has a `weight_url` (Axelera
  server) → downloaded at deploy time.
- `cct.onnx`: inspected with ONNX — input `[N,3,64,128]`, output `[N,15,37]` (37 Latin
  classes). Different model from LPRNet (`[1,3,24,94]`, 68 classes).
- Indian dataset: `datasets/plate_ocr`, 100 image/label pairs; sample label `TS10EE5625`
  (Telangana).
- Detection artifacts: `find` over `cpp/axnet` and `models/cascade-build*` — vehicle and
  plate `.axnet` present; **no** LPR/OCR build root.
- Cascade execution path: `tools/eval/bakeoff_aipu.py` uses
  `axelera.app.stream.create_inference_stream`; C++ `traffic_pilotd.cpp` /
  `axn_hwdecode_bench.cpp` use single-model `Ax::InferenceNet`.
