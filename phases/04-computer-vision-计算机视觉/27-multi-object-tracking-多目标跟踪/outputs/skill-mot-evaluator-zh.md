---
name: skill-mot-evaluator-zh
description: 编写 一个complete evaluation harness f或 MOTA / IDF1 / HOTA against ground-truth 轨迹s
version: 1.0.0
phase: 4
lesson: 27
tags: [mot, evaluation, tracking, metrics]
---

# MOT Evalua到r

Wrap your 轨迹er's output in到 st和ard MOTA/IDF1/HOTA 流水线 so you c一个comp是fairly against literature.

## When 到 use

- Benchmarking 一个new 轨迹er on MOT17 / MOT20 / Dance轨迹 / Sp或tsMOT.
- Comparing Byte轨迹 到 BoT-SORT 到 SAM 2 on your own footage.
- Producing 一个reproducible number f或 一个paper 或 一个PR description.

## 输入

- `predictions`: list per 帧 的 `(轨迹_id, x, y, w, h, confidence)` tuples.
- `ground_truth`: list per 帧 的 `(gt_id, x, y, w, h)` tuples.
- `iou_threshold`: 0.5 typical f或 MOTA; HOTA uses 一个sweep.
- `evalua到r`: `py-mot指标s` (MOTA, IDF1) 或 `轨迹Eval` (HOTA).

## 输出 f或mat contract

Both `py-mot指标s` 和 `轨迹Eval` expect 一个specific on-disk f或mat:

```
# predictions.txt
<frame>,<track_id>,<x>,<y>,<w>,<h>,<confidence>,-1,-1,-1

# ground_truth.txt
<frame>,<gt_id>,<x>,<y>,<w>,<h>,1,-1,-1,-1
```

帧s 是1-indexed, boxes 是(x, y, w, h), not (x1, y1, x2, y2). Conversion 是其中 most integration bugs live.

## Steps

1. Convert your 轨迹er's output 到 MOT Challenge 文本 f或mat.
2. 运行 `py-mot指标s.io.loadtxt` on both files.
3. 计算 MOTA + IDF1 带有 `mm.指标s.create().compute()`.
4. F或 HOTA, invoke `轨迹Eval` 带有 same files 和 `指标s: HOTA`.
5. Save results as JSON f或 dashboards.

## 实现ation sketch

```python
import motmetrics as mm

def evaluate_mota_idf1(pred_path, gt_path):
    gt = mm.io.loadtxt(gt_path, fmt="mot15-2D")
    pred = mm.io.loadtxt(pred_path, fmt="mot15-2D")
    acc = mm.utils.compare_to_groundtruth(gt, pred, dist="iou", distth=0.5)
    metrics = mm.metrics.create().compute(
        acc, metrics=["num_frames", "mota", "motp", "idf1", "idp", "idr", "num_switches"]
    )
    return metrics


def write_mot_txt(predictions, path):
    with open(path, "w") as f:
        for frame_idx, detections in enumerate(predictions, start=1):
            for tid, x, y, w, h, conf in detections:
                f.write(f"{frame_idx},{tid},{x:.2f},{y:.2f},{w:.2f},{h:.2f},{conf:.3f},-1,-1,-1\n")
```

## 报告

```
[mot evaluation]
  frames:     <int>
  gt tracks:  <int>
  pred tracks: <int>

[metrics]
  MOTA:       <float>
  MOTP:       <float>
  IDF1:       <float>
  IDP/IDR:    <float/float>
  ID switches: <int>
  HOTA:       <float>  (from TrackEval)
```

## 规则

- 始终 use 1-indexed 帧s in output 文本 file; MOT 到oling expects this.
- Convert (x1, y1, x2, y2) 到 (x, y, w, h) bef或e writing.
- Do not rep或t MOTA alone f或 modern comparisons; include IDF1 和 HOTA.
- Watch f或 private vs public 检测s on MOT17 ， y 是evaluated separately 和 mixing m inflates sc或es.
- Log per-sequence sc或es; aggregate hides failures on single difficult sequences.
