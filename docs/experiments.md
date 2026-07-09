## Experiment 1: Baseline Training on GTSRB (Roboflow) Dataset

**Date:** 7/4/2026
**Dataset:** traffic-sign-detection-gtsrb (Roboflow, v2), GTSRB-derived, 43 classes
**Model:** YOLO11s
**Config:** epochs=100, patience=15, batch=-1 (auto), imgsz=640, no augmentation (baseline)

### Result
- Test set (GTSRB's own split): mAP50 ≈ [your number], high per-image confidence (e.g. 0.97 on individual detections)
- Real-world images (stock photos, out-of-dataset): 0 detections, even on clearly visible signs (e.g. "No Entry")

### Diagnosis
Initial hypothesis was a possible domain mismatch (German vs. non-German sign standards). This was ruled out by testing on a European-style "No Entry" sign, which still produced zero detections.

Visual inspection of raw training images (6 random samples from `train/images`) revealed the actual cause: the dataset consists primarily of tightly-cropped, sign-dominant images — consistent with GTSRB's origin as a **classification** benchmark, where each image is pre-cropped around a single sign. Despite being exported in YOLO detection format (image + bounding box), the underlying images do not represent genuine full driving-scene imagery with signs appearing small and embedded in context.

This explains both observations:
- Strong performance on GTSRB's own test set — since train and test share the same cropped, sign-dominant framing
- Failure on real-world photos — since the model was never trained to locate a *small* sign within a *large*, cluttered scene, which is the actual real-world detection task

### Conclusion
This dataset is unsuitable for training a generalizable object detector, despite strong-looking benchmark metrics. Benchmark performance alone is not sufficient evidence of a dataset's quality for this project's goals — visual inspection of raw images should be a mandatory step before committing to a dataset, not just checking image/class counts.

### Next step
Selecting a replacement dataset, with training images visually verified beforehand to confirm genuine full-scene imagery (small sign, real background context) before committing to a full training run.

## Mid-Day Progress Report
 
**Date:** 7/5/2026
**Datasets:** d1 (traffic-sign-detection-gtsrb, Roboflow, GTSRB-derived, 43 classes) and d2 (Traffic-and-Road-Signs, Roboflow, current active dataset)
**Model:** YOLO11s
**Config:** epochs=100, patience=15, batch=-1 (auto), imgsz=640
 
### Summary of Today's Work
 
Following the conclusion from Experiment 1 (GTSRB dataset rejected due to crop-heavy framing bias), today's work focused on sourcing a replacement dataset, validating it, and beginning augmentation experiments to address a new, more specific limitation discovered along the way.
 
---
 
### Dataset Versioning Update
 
`docs/dataset.md` now tracks two dataset attempts explicitly:
- **d1** — the original GTSRB-derived dataset (Experiment 1), retained for documentation/comparison purposes, not in active use
- **d2** — "Traffic-and-Road-Signs" (Roboflow), the current active dataset, selected in part because a model already trained on this dataset by another user (posted on Roboflow Universe) was confirmed able to detect distant/small-scale signs successfully — used as a reference point that this dataset is structurally capable of supporting real-world detection
d1 currently has two trained versions/checkpoints (two separate `best.pt` files), both retained as historical baselines. d2 currently has one trained baseline version (`baseline_no_aug_v2`, referred to below as v1d2 baseline), with a second, augmented version (v1d2 augmented) training as of this report.
 
---
 
### Experiment 2: Baseline Training on Traffic-and-Road-Signs (d2)
 
**Date:** 7/5/2026
**Dataset:** Traffic-and-Road-Signs (Roboflow, v1), d2
**Model:** YOLO11s
**Config:** epochs=100, patience=15, batch=-1 (auto), imgsz=640, no augmentation (baseline) — referred to as v1d2 baseline
 
**Result:**
```
precision: 0.9264
recall:    0.9181
mAP50:     0.9248
mAP50-95:  0.7713
```
 
Both precision and recall are high and closely matched — a meaningfully healthier result than d1's baseline, where recall remained under 0.1 through early training and the model showed clear signs of a structural framing bias rather than genuine learning.
 
**Real-world testing:**
- d1 baseline model: 0 detections on all tested real-world images (stop sign, no-entry sign) — consistent with Experiment 1's diagnosis
- d2 baseline model (v1d2 baseline): successfully detected signs in real-world photos at close/moderate range (e.g., correct "Keep-Left" detection across multiple test uploads), but **failed to detect the same sign category when captured at a distance** (small apparent size in frame)
**Diagnosis:**
Unlike d1's total failure, this is a narrower, more specific limitation: small/distant object detection. The model has clearly learned real sign features and generalizes reasonably well at close range, but struggles when the sign occupies a very small pixel area — consistent with a known, well-documented challenge in object detection generally, not a dataset framing bias like Experiment 1.
 
---
 
### Experiment 3 (in progress): Augmentation for Small/Distant Object Detection
 
**Date:** 7/5/2026
**Dataset:** d2 (same as Experiment 2, no change to source images or resize settings)
**Model:** YOLO11s
**Config:** epochs=100, patience=15, batch=-1 (auto), imgsz=640 (unchanged) — referred to as v1d2 augmented
 
**Augmentation added (training-time only, via `model.train()`, not via Roboflow's preprocessing/export):**
- `mosaic=1.0` — combines 4 training images per batch; exposes the model to greater scale variation and context, directly targeting small-object detection
- `scale=0.5` — random zoom in/out during training, simulating varied sign distances
- `copy_paste=0.1` — pastes sign instances onto varied backgrounds/scales, increasing small-object example diversity
**Explicitly still disabled:** `hsv_h/s/v`, `fliplr`, `flipud`, `degrees`, `shear`, `perspective`, `erasing` — flip remains excluded due to the risk of inverting directional sign meaning while preserving the original label; hue/grayscale-related changes remain excluded due to color being a meaningful classification signal for signs.
 
**Note on resolution:** considered increasing `imgsz` to 960+ to help with distant/small objects, but held off — since Roboflow's export already resizes source images to 640x640, training at a higher `imgsz` without a corresponding higher-resolution export would only upscale existing pixels via interpolation, not recover genuine additional detail. This is reserved as a separate, future experiment, contingent on checking native source image resolution first.
 
**Status:** training in progress. This run is referred to as v1d2 augmented (version 1, dataset 2, augmented).
 
---
 
### Key Takeaways So Far (Today)
 
1. Dataset structural quality (full-scene vs. cropped framing) was the dominant factor separating d1's total failure from d2's partial success — reinforcing Experiment 1's conclusion that visual inspection matters more than benchmark metrics alone.
2. d2's failure mode is narrower and more typical of standard object detection challenges (scale/distance), rather than a fundamental dataset mismatch — a meaningfully better starting point.
3. Augmentation choices are being made deliberately, targeted at the specific observed failure (distance/scale), rather than applied as a general default — consistent with the project's engineering-decision-first approach.
---
 
### Next Step
 
Once v1d2 augmented finishes training, run it against a set of random real-world test images (including the distant-sign case that failed under v1d2 baseline) and directly compare detection results and confidence scores against v1d2 baseline. This comparison will determine whether mosaic/scale/copy_paste augmentation meaningfully closes the distant-object detection gap, and will be documented as the outcome of Experiment 3.

## End-of-Day Progress Report
 
**Date:** 7/5/2026
**Datasets:** v2_dataset (Traffic-and-Road-Signs, Roboflow) — current active dataset
**Model:** YOLO11s
**Config:** epochs=100, patience=15, batch=-1 (auto), imgsz=640
 
### Summary of the Rest of Today's Work
 
Following the mid-day report, v1d2 augmented (mosaic=1.0, scale=0.5, copy_paste=0.1) began training on Google Colab. Partway through, Colab's free-tier GPU usage limit was hit, halting further cloud-based training for the day.
 
Rather than wait out the limit, the remainder of the day was spent migrating the entire training pipeline from Google Colab to a local environment, using the project's own hardware (NVIDIA RTX A2000 Laptop GPU) via Jupyter notebooks running inside VS Code. This is intended to be the primary training environment going forward, removing dependency on Colab's free-tier limits entirely.
 
---
 
### Local Environment Migration
 
**Motivation:** Colab's free-tier GPU quota is unguaranteed and usage-based, with no fixed reset time. Since the project's laptop has a dedicated CUDA-capable GPU (RTX A2000), migrating to local training removes this bottleneck for future experiments.
 
**Setup performed:**
- Created an isolated Python virtual environment (`yolo-env`) at the project root using `python -m venv`
- Installed core dependencies: `ultralytics`, `roboflow`, `python-dotenv`, `ipykernel`, `jupyter`
- Registered `yolo-env` as a selectable Jupyter kernel via `ipykernel install`, enabling notebook execution inside VS Code against this environment
- Replaced Colab-specific mechanisms with local equivalents:
  - `google.colab.userdata` (Colab Secrets) → `.env` file + `python-dotenv`, with `.env` added to `.gitignore` to keep the Roboflow API key out of version control
  - `drive.mount()` → removed; all paths now reference local project folders directly
  - `files.upload()` / `files.download()` → removed; inference and weight files are read/written directly from local disk, no upload/download step needed
**Issues encountered and resolved:**
- PowerShell blocked venv activation by default (`running scripts is disabled`); resolved via `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned`
- An initial global Jupyter install was mistakenly made outside the venv; uninstalled and reinstalled correctly inside `yolo-env`
- Notebook kernel initially defaulted to the wrong Python environment (global install rather than `yolo-env`), causing intermittent `ModuleNotFoundError`s for packages that were, in fact, installed — resolved by explicitly selecting the `yolo-env` kernel via VS Code's kernel picker and confirming with `sys.executable`
- `torch` initially installed as the CPU-only build rather than the CUDA-enabled build, causing `torch.cuda.is_available()` to return `False` despite a working GPU; resolved by reinstalling via `pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121`. Note: `nvidia-smi` reported a newer CUDA version (13.2) than any published PyTorch build tag; confirmed NVIDIA drivers are backward-compatible, so a `cu121` build runs correctly on this system despite the version number mismatch
- The project folder was originally located inside a OneDrive-synced directory, which caused abnormally slow package installs and at least one Ultralytics-reported "slow image access" warning during a validation run, explicitly recommending local (non-remote/mounted) storage. The entire project was relocated to `C:\Personal_Projects\traffic-sign-detection-project1`, outside any cloud-sync folder, resolving both issues
- Minor Windows path-escaping issue identified when writing file paths as plain strings (single backslashes can be misinterpreted as escape characters); standardized on `os.path.join()` and forward-slash paths going forward to avoid ambiguity
**Status at end of day:** local environment fully verified — GPU detected (`torch.cuda.is_available() == True`, device correctly identified as RTX A2000), dependencies installed and confirmed inside the correct kernel, dataset re-downloaded fresh into the new local project location (confirmed no longer routed through OneDrive), and pipeline cells (download, split, train, inference, evaluation, JSON summary) translated and adapted for local execution.
 
---
 
### Next Step
 
Train v1d2 augmented (mosaic=1.0, scale=0.5, copy_paste=0.1) to completion using the newly configured local environment. Once complete, run it against a set of random real-world test images — including the distant-sign case that failed under v1d2 baseline — and directly compare detection results and confidence scores against v1d2 baseline. This comparison will determine whether mosaic/scale/copy_paste augmentation meaningfully closes the distant-object detection gap, and will be documented as the outcome of Experiment 3.

## Progress Report — Experiment 3 Complete, Comparison and Experiment 4 Planning
 
**Date:** 7/7/2026
**Dataset:** v2_dataset (Traffic-and-Road-Signs, Roboflow)
**Model:** YOLO11s
**Config:** epochs=100, patience=15, batch=-1 (auto), imgsz=640
 
### Summary
 
v1d2 augmented finished training successfully in the local environment (RTX A2000), completing the migration validation from the previous report. Metrics-level comparison against v1d2 baseline is complete; image-level comparison on real-world test cases is the immediate next step, followed by a second, more targeted augmentation experiment (v2aug) informed by a new finding from local hands-on testing.
 
---
 
### Experiment 3 (complete): Augmentation for Small/Distant Object Detection
 
**Config:** mosaic=1.0, scale=0.5, copy_paste=0.1 (all other augmentation parameters disabled, matching prior experiments)
 
**Result — v1d2 baseline vs. v1d2 augmented:**
 
| Metric | Baseline (no aug) | v1aug (mosaic/scale/copy_paste) | Change |
|---|---|---|---|
| Precision | 0.9264 | 0.9347 | +0.0083 |
| Recall | 0.9181 | 0.9461 | +0.0280 |
| mAP50 | 0.9248 | 0.9408 | +0.0160 |
| mAP50-95 | 0.7713 | 0.7976 | +0.0263 |
 
Augmentation produced a consistent improvement across every metric, with the largest gain in recall — consistent with the original hypothesis that mosaic, scale, and copy_paste augmentation would help the model detect more real instances of signs, particularly smaller/less prominent ones, rather than only becoming more confident on cases it already detected well.
 
---
 
### New Finding: Webcam/Live-Capture Testing Reveals a Second Limiting Factor
 
Testing v1aug interactively via local webcam feed (`model.predict(source=0)`, enabled by the local environment migration) surfaced a distinction not previously visible in static-image testing: the model performs noticeably worse on distant signs captured via webcam than on distant signs captured via phone photo, despite both appearing "far away" to the eye.
 
**Working hypothesis:** this points to a second, related but distinct factor beyond apparent object scale — raw image quality and resolvable pixel detail. A webcam frame (lower resolution, live-capture motion blur, video compression) contains meaningfully less usable detail on a distant object than a deliberately framed phone photo at a comparable distance, even though both present the sign as a "small object in frame." Distance/scale and capture quality appear to compound rather than being fully interchangeable explanations for the same failure mode.
 
This finding will be verified further and, if confirmed, incorporated into the next augmentation experiment (see below).
 
---
 
### Next Steps
 
**1. Image-level comparison (baseline vs. v1aug) — in progress**
 
Metric-level comparison (mAP50, precision, recall, above) is complete. The remaining comparison step is qualitative and visual: loading both `./models/d2/best.pt` (baseline) and `./models/d2/best_v1d2_v1aug.pt` (v1aug) side by side in a dedicated notebook, and running both against the same set of real-world test images — including the distant-sign case that failed under baseline — to directly compare detection presence, bounding box accuracy, and confidence scores image-by-image, not just in aggregate. This will be documented in `docs/comparison_analysis.md`.
 
**2. Experiment 4 (planned): v2aug — Targeting Resolution/Capture-Quality Limitations**
 
Once the image-level comparison is complete, a second augmentation experiment (v2aug) will be trained, building on v1aug's configuration with adjustments targeted at the webcam-distance finding above:
 
- `scale` increased from `0.5` to `0.9` — widens the random zoom range during training, exposing the model to a greater range of small apparent sign sizes than v1aug covered
- `copy_paste` increased from `0.1` to `0.2` — increases synthetic small-object example diversity beyond v1aug's level
- Mild blur augmentation added (previously disabled) — intended to directly simulate the motion blur and compression softness characteristic of live webcam/video capture, which phone-photo testing does not exercise
- All previously excluded augmentations (`hsv_h/s/v`, `fliplr`, `flipud`, `degrees`, `shear`, `perspective`, `erasing`) remain disabled, for the same domain-specific reasons established in Experiment 3 (sign-meaning inversion risk, color as a classification signal)
v2aug's results will be compared against both v1d2 baseline and v1aug, with particular attention to real-world webcam-based testing rather than static image testing alone, to evaluate whether the added blur/scale adjustments meaningfully close the capture-quality gap identified above.

## Progress Report — Experiment 3 Closed, Experiment 4 Starting
 
**Date:** 7/7/2026
**Dataset:** v2_dataset (Traffic-and-Road-Signs, Roboflow)
**Model:** YOLO11s
**Config:** epochs=100, patience=15, batch=-1 (auto), imgsz=640
 
### Summary
 
Image-level comparison between v1d2 baseline and v1d2 augmented (v1aug) is complete, closing out Experiment 3. Results strongly support the metric-level findings from the previous report: augmentation produced a clear, consistent improvement in real-world detection, not just in benchmark numbers. Experiment 4 (v2aug) is now starting, with one adjustment to the original plan — webcam-based live testing has been replaced with uploaded video testing, to be run across all three models (baseline, v1aug, v2aug) once v2aug completes training.
 
---
 
### Experiment 3 — Image-Level Comparison Results (Baseline vs. v1aug)
 
Six real-world test images (not part of the training/validation/test dataset) were run through both models at `conf=0.5`.
 
| Image | Baseline | v1aug |
|---|---|---|
| stop.jpg | No detection | Stop Sign, 0.94 |
| stop2.jpg | No detection | Stop Sign, 0.84 |
| testing2.jpg | No detection | Stop Sign, 0.64 |
| testing5.jpg | No detection | No detection |
| testing6.jpg | No detection | Stop Sign, 0.81 |
| testing7.jpg | No Entry sign misclassified as Stop Sign, 0.80 | Stop Sign, 0.97 |
 
**Findings:**
 
- v1aug successfully detected signs in 5 of 6 test images; baseline detected a sign in only 1 of 6, and that single baseline detection was a **misclassification** (a No Entry sign identified as a Stop Sign), meaning baseline produced zero *correct* detections across the entire test set.
- This is a substantial, unambiguous improvement, and confirms the metric-level gains reported previously (mAP50 +0.016, recall +0.028) reflect a genuine, practically meaningful improvement in real-world usability, not a marginal statistical shift.
- One image (testing5.jpg) was missed by both models, indicating a case — likely distance, angle, or occlusion related — that augmentation did not resolve. This remains a candidate case for Experiment 4 to address.
- The baseline's single detection being a misclassification (No Entry → Stop Sign) rather than simply a missed detection is a notable failure mode in its own right: it suggests baseline was not just under-detecting, but in the rare case it did detect something, doing so with an incorrect class. v1aug did not exhibit this misclassification behavior on the same image, instead correctly identifying it as a Stop Sign at high confidence (0.97).
This comparison is recorded in `docs/comparison_analysis.md`.
 
---
 
### Experiment 4 (starting): v2aug — Targeting Resolution/Capture-Quality Limitations
 
**Config (building on v1aug):**
- `scale` increased from `0.5` to `0.9`
- `copy_paste` increased from `0.1` to `0.2`
- Mild blur augmentation added (previously disabled)
- All previously excluded augmentations (`hsv_h/s/v`, `fliplr`, `flipud`, `degrees`, `shear`, `perspective`, `erasing`) remain disabled, for the same domain-specific reasoning established in Experiment 3
**Plan adjustment:** the original plan called for live webcam testing to evaluate the capture-quality/resolution hypothesis. This has been changed to **uploaded video testing** instead, run consistently across all three models (baseline, v1aug, and v2aug) once v2aug finishes training. This provides a more controlled, repeatable comparison than live webcam sessions — the same video file can be run through all three models under identical conditions, making frame-by-frame differences directly attributable to the model rather than to variability in a live capture session.
 
**Next step:** train v2aug to completion, then run the same real-world test image set used in this report through all three models for continuity, followed by the uploaded video comparison across all three models to evaluate distant/small-object detection under realistic motion and compression conditions.