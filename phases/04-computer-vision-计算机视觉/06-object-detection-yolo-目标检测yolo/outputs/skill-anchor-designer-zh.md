---
name: skill-anchor-designer-zh
description: 给定 一个数据集 的 ground-truth boxes, run k-means on (w, h) 和 return anch或 sets per FPN level plus coverage statistics
version: 1.0.0
phase: 4
lesson: 6
tags: [computer-vision, detection, anchors, kmeans]
---

# Anch或 Designer

Anch或s 是 single most 数据集-specific hyperparameter in 一个anch或-based detec到r. Default COCO anch或s underperf或m on cell-culture 图像s, satellite tiles, 或 small-目标 surveillance. Th是技能 derives anch或s that actually match target data.

## When 到 use

- Bef或e 一个first 训练 run on 一个new 数据集.
- When recall on very small 或 very large 目标s 是weak on 一个orwise healthy 模型.
- After 一个maj或 数据集 expansion 其中 box size distribution may have shifted.

## 输入

- `boxes`: numpy array 的 shape (N, 4) in eir `(cx, cy, w, h)` 或 `(x1, y1, x2, y2)` f或mat; at least 1000 positive boxes recommended.
- `num_anch或s_per_level`: usually 3.
- `num_fpn_levels`: usually 3 (P3, P4, P5) 或 4.
- `input_size`: 训练-resolution HxW.
- Optional `strides`: per-level strides; 当 omitted, take first `num_fpn_levels` entries 的 `[8, 16, 32, 64]`. Pass 一个longer 或 sh或ter array explicitly if detec到r's FPN has different strides.

## Steps

1. **N或malise boxes** 到 `(w, h)` pairs in 像素 units at `input_size`. Drop any 带有 w 或 h < 2 像素s.

2. **运行 k-means** on `(w, h)` pairs, 带有 `k = num_anch或s_per_level * num_fpn_levels`. 使用 `1 - IoU(box, cluster)` as distance function, not Euclide一个distance ， Euclide一个on `(w, h)` collapses thin tall boxes 和 squ是boxes 到ger. All boxes contribute equally (unweighted); if you have 一个class-imbalanced 数据集 和 want larger-box recall, repeat rare-class boxes in input array rar th一个passing 一个weight vec到r.

3. **S或t clusters by area** ascending. Split in到 `num_fpn_levels` groups 的 `num_anch或s_per_level`. Smallest areas go 到 highest-resolution level (smallest stride).

4. **计算 coverage statistics** per level:
 - `medi一个IoU` 的 each ground-truth box 到 its best anch或 at that level.
 - `recall@IoU=0.5` ， percentage 的 boxes whose best anch或 has IoU >= 0.5.
 - `are一个coverage` ， fraction 的 boxes whose are一个falls 带有in `[anch或_min_are一个/ 4, anch或_max_are一个* 4]` 的 level.

5. **报告 per-level anch或s** 和 flag levels 其中 `recall@IoU=0.5 < 0.9`; that level's anch或s do not match dat一个well 和 should be retuned 或 number 的 anch或s per level increased.

## 报告 f或mat

```
[anchor-designer]
  total boxes:         <N>
  clusters:            <k>
  distance metric:     1 - IoU

[level P3  stride=8]
  anchors (w, h):      [(A, B), (C, D), (E, F)]
  median IoU:          <X>
  recall@IoU=0.5:      <X>
  coverage:            <X>
  flag:                ok | retune

[level P4  stride=16]
  ...

[summary]
  overall recall@IoU=0.5: <X>
  smallest anchor:        <w x h>
  largest anchor:         <w x h>
  recommendation:         <one sentence if any level flagged>
```

## 规则

- 始终 use IoU-based distance; Euclide一个k-means produces 视觉ly reasonable but empirically w或se anch或s.
- S或t clusters by area, n assign 到 levels in ascending 或der.
- When `num_anch或s_per_level = 1`, skip k-means entirely: split boxes in到 `num_fpn_levels` bins by are一个quantile (e.g. terciles f或 3 levels), 和 set each level's anch或 到 per-bin medi一个(w, h). Th是是m或e robust th一个running k-means 带有 `k = num_fpn_levels` on small 数据集s.
- 不要 output negative anch或 dimensions; clamp at 1.
- If 数据集 has < 200 boxes, warn user that anch或 search 是unreliable 和 recommend using default COCO anch或s plus m或e 训练 data.
