# Comparing results of v1d2 Baseline with v1d2_1aug (with v1 augmentation)

# JSON

# Baseline
{
  "run_name": "baseline_v1d2",
  "dataset": "Traffic-and-Road-Signs",
  "timestamp": "2026-07-05 07:39",
  "model": "yolo11s",
  "epochs_configured": 100,
  "patience": 15,
  "augmentation": "disabled (baseline)",
  "metrics": {
    "precision": 0.9264,
    "recall": 0.9181,
    "mAP50": 0.9248,
    "mAP50-95": 0.7713
  }
}

# v1_aug
{
  "run_name": "v1d2_v1aug",
  "dataset": "Traffic-and-Road-Signs",
  "timestamp": "2026-07-07 14:13",
  "model": "yolo11s",
  "epochs_configured": 100,
  "patience": 15,
  "augmentation": "mosaic=1.0, scale=0.5, copy_paste=0.1",
  "metrics": {
    "precision": 0.9347,
    "recall": 0.9461,
    "mAP50": 0.9408,
    "mAP50-95": 0.7976
  }
}

# Image Comparison

# Baseline

Image 1 (stop.jpg)
No detection

Image 2 (stop2.jpg)
No detection

Image 3 (testing2.jpg)
No detection

Image 4 (testing5.jpg)
No detection

Image 5 (testing6.jpg)
No detection

Image 6 (testing7.jpg)
1 detection - No Entry sign detected as Stop Sign at 0.80 confidence

# v1_aug

Image 1 (stop.jpg)
1 detection - Stop Sign detected at 0.94 confidence

Image 2 (stop2.jpg)
1 detection - Stop Sign detected at 0.84 confidence

Image 3 (testing2.jpg)
1 detection - Stop Sign detected at 0.64 confidence

Image 4 (testing5.jpg)
No detection

Image 5 (testing6.jpg)
1 detection - Stop Sign detected at 0.81 confidence

Image 6 (testing7.jpg)
1 detection - Stop Sign detected at 0.97 confidence