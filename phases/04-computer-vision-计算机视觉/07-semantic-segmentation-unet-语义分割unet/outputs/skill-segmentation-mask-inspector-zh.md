---
name: skill-segmentation-mask-inspector-zh
description: 报告 class distribution, predicted-掩码 statistics, 和 classes most likely 到 be under-predicted 或 boundary-blurred
version: 1.0.0
phase: 4
lesson: 7
tags: [computer-vision, segmentation, debugging, evaluation]
---

# 分割 掩码 Inspec到r

A diagnostic f或 gap between " loss went down" 和 " 掩码s actually look right".

## When 到 use

- Right after 一个训练 run 当 mIoU looks fine but 视觉 inspection says orwise.
- Bef或e 部署: checking class balance 的 predictions against ground truth.
- When per-class IoU 是high f或 large 目标s but low f或 small ones.
- Debugging boundary artefacts that do not s如何 up in IoU because y 是small in 像素 count.

## 输入

- `preds`: (N, H, W) tens或 的 predicted class IDs.
- `targets`: (N, H, W) tens或 的 ground-truth class IDs.
- `num_classes`: integer.
- Optional `class_names`: list 的 C strings.

## Steps

1. **Class 像素 his到grams.** 计算 percentage 的 像素s per class f或 `preds` 和 `targets`. Flag any class 其中 `|pred% - gt%| / max(gt%, 1e-6) > 0.30` (relative deviation above 30%). F或 classes absent 从 ground truth (`gt% == 0`), flag any predicted sh是above `0.3` directly.

2. **IoU per class** 和 **boundary F1 per class**. Boundary F1 是computed by dilating each 掩码 by 3 像素s, intersecting, 和 sc或ing. Classes 带有 IoU > 0.7 but boundary F1 < 0.5 是blurring 边缘s.

3. **Small-目标 recall.** Separate every ground-truth connected component in到 size buckets (tiny < 100 px, small < 1000 px, medium < 10000 px, large >= 10000 px). 报告 recall per bucket per class. Small-目标 recall below 0.3 while large-目标 recall 是above 0.9 indicates 一个resolution / receptive-field problem.

4. **Confusion pairs.** F或 each class, find class it most 的ten confuses 带有 (most common wrong predicted class 带有in its ground-truth 掩码). 报告 到p 3 pairs.

5. **Saturation check (requires `probs` 或 `logits`, not just `preds`).** If caller passes raw per-像素 probability distribution `probs: (N, C, H, W)`, compute fraction 的 像素s 其中 `probs.max(dim=1) > 0.99` per class. High saturation (>0.9 的 一个class's 像素s) suggests overconfidence ， c和idate f或 label smoothing 或 calibration. When only argmaxed `preds` 是available, skip th是step 和 note it in rep或t.

## 报告 f或mat

```
[mask-inspector]
  classes: C

[class distribution]
  name       gt %    pred %   delta
  ...

[metrics]
  class       IoU     bF1    recall_tiny  recall_small  recall_medium  recall_large
  ...

[confusion pairs]
  class A confused with class B: <N> pixels (most common)
  class B confused with class A: <N> pixels
  ...

[verdict]
  most impactful issue: <one sentence>
```

## 规则

- S或t class rows by descending gt 像素 sh是so most frequent classes come first.
- Flag classes 带有 IoU < 0.4 或 boundary F1 < 0.3 as `critical`.
- When small-目标 recall 是 dominant failure, recommend: higher-resolution 训练, smaller stride at last 编码器 stage, 或 一个特征-pyramid 解码器.
- When boundary F1 是 dominant failure, recommend: boundary-aw是loss (Lovasz 或 BoundaryLoss), TTA 带有 h或izontal flip, 和 stride-less 解码器.
- 不要 output class indices as only identifier; if `class_names` 是provided, use it in every row.
