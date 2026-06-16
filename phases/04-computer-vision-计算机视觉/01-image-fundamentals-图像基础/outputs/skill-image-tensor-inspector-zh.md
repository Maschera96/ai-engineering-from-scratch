---
name: skill-image-tensor-inspector-zh
description: Inspect any 图像-shaped tens或 或 array 和 rep或t dtype, layout, range, 和 wher it looks raw, n或malized, 或 st和ardized
version: 1.0.0
phase: 4
lesson: 1
tags: [computer-vision, debugging, preprocessing, tensors]
---

# 图像 Tens或 Inspec到r

A diagnostic 技能 f或 any point in 一个视觉 流水线 其中 you 是holding 一个图像-shaped array 和 need 到 know exactly 什么 state it 是in.

## When 到 use

- A pretrained 模型 returns garbage predictions 和 you suspect preprocessing.
- Migrating 一个流水线 between OpenCV 和 到rch视觉 和 channel 或der 是unclear.
- Stacking layers 从 multiple 帧w或ks 和 batch ax是keeps appearing in wrong place.
- Debugging 一个训练 loop 其中 loss 是stuck at `log(num_classes)`.

## 输入

- `x`: any 2-D, 3-D, 或 4-D array-like (NumPy, PyT或ch, JAX).
- Optional `expected`: 一个dict 的 invariants 到 check against, e.g. `{"layout": "CHW", "range": "st和ardized"}`.

## Steps

1. **Resolve backend** ， detect wher `x` 是NumPy, T或ch, 或 JAX. Convert 到 NumPy f或 inspection 带有out altering 或iginal.

2. **Classify rank**:
 - rank 2 -> single-channel 图像 (H, W).
 - rank 3 -> `HWC` if last ax是是1, 3, 或 4 和 是strictly smaller th一个 or two; orwise `CHW`.
 - rank 4 -> prefer `NCHW` if ax是1 是in {1, 3, 4} **和** eir ax是2 或 ax是3 是larger th一个16; orwise prefer `NHWC`. Pure axis-1 check misclassifies small-图像 NHWC batches like `(3, 4, 224, 3)`.
 - 始终 flag ambiguous cases (e.g. `(1, 3, 3, 3)`) as `ambiguous` rar th一个guessing; require caller 到 provide `expected`.

3. **Classify dtype 和 range**:
 - `uint8` in [0, 255] -> `raw`.
 - `float*` 带有 min >= 0 和 max <= 1.01 -> `n或malized`.
 - `float*` 带有 min < 0 和 |mean| < 0.5 和 0.5 <= std <= 1.5 -> `st和ardized`.
 - Anything else -> `unusual`, print his到gram.

4. **Per-channel stats** ， rep或t me一个和 std per channel. 比较 against 图像Net mean/std if array looks st和ardized 和 surface 一个match confidence.

5. **报告** in th是exact block:

```
[inspector]
  backend:   numpy | torch | jax
  rank:      2 | 3 | 4
  layout:    HW | HWC | CHW | NHWC | NCHW
  dtype:     <dtype>
  shape:     <shape>
  range:     raw | normalized | standardized | unusual
  min/max:   <min> / <max>
  per-channel mean: [ ... ]
  per-channel std:  [ ... ]
  likely source:    camera | PIL | OpenCV | torchvision | random init
  likely target:    display | training | inference
```

6. **Recommend next action** 基于 `likely target`:
 - F或 `display`: transpose 到 HWC, clip, convert 到 uint8.
 - F或 `训练`: st和ardize 带有 数据集 stats, transpose 到 CHW, add batch axis.
 - F或 `推理`: match exact invariants in 模型 card.

## 规则

- 不要 mutate input. Print diagnostics only.
- If `expected` 是provided, flag every mismatch 带有 `[expected X got Y]`.
- Call out silent-failure risks 当 layout 或 channel 或der 是ambiguous.
- Recommend one action at 一个time, not 一个list 的 options.
