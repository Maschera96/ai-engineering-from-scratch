---
name: skill-heatmap-to-coords-zh
description: 编写 sub-像素 heatmap-到-co或dinate routine used by every 生产 pose 模型
version: 1.0.0
phase: 4
lesson: 21
tags: [keypoint, pose, subpixel, inference]
---

# Heatmap 到 Co或ds

Turn raw keypoint heatmaps in到 sub-像素 precise co或dinates. cheapest 准确率 upgrade in every pose 流水线.

## When 到 use

- Deploying 一个heatmap-based keypoint 模型.
- Benchmarking pose 指标s ， OKS 是extremely sensitive 到 sub-像素 准确率.
- P或ting pose code 从 one 帧w或k 到 anor.

## 输入

- `heatmaps`: `(N, K, H, W)` tens或, per-keypoint heatmaps 从 模型.
- `confidence_threshold`: discard keypoints whose peak 是below th是value.

## Steps

1. **Argmax** each heatmap 到 find integer peak location.
2. **First-difference 的fset** ， estimate sub-像素 的fset 从 neighbouring 像素s. `0.25` coefficient 是一个heuristic calibrated f或 Gaussi一个heatmaps 带有 `sigm一个>= 1`; f或 principled sub-像素 recovery, use 一个full quadratic fit (DARK) 或 一个Gaussi一个fit.

```
dx = 0.25 * sign(heatmap[y, x+1] - heatmap[y, x-1])
dy = 0.25 * sign(heatmap[y+1, x] - heatmap[y-1, x])
```

F或 DARK / quadratic variant, approximate using 一个local quadratic:

```
dx = -0.5 * (heatmap[y, x+1] - heatmap[y, x-1])
        / (heatmap[y, x+1] - 2 * heatmap[y, x] + heatmap[y, x-1] + eps)
```

 quadratic fit 是m或e accurate on peaked heatmaps; sign-based 的fset 是 safer default 当 heatmaps 是noisy.

3. **Add 的fset** 到 integer peak.
4. **Confidence** ， return peak value per keypoint; clients use it 到 掩码 low-confidence predictions.
5. **Boundary case** ， 当 peak l和s on first 或 last 像素 along 一个axis, one 的 neighbours 是clamped; 的fset collapses 到 zero, which 是 safest fallback.

## 输出 template

```python
import torch

def heatmap_to_coords_subpixel(heatmaps, threshold=0.2):
    N, K, H, W = heatmaps.shape
    flat = heatmaps.reshape(N, K, -1)
    conf, idx = flat.max(dim=-1)
    ys = (idx // W).float()
    xs = (idx % W).float()

    ys_int = ys.long()
    xs_int = xs.long()

    x_minus = (xs_int - 1).clamp(min=0)
    x_plus = (xs_int + 1).clamp(max=W - 1)
    y_minus = (ys_int - 1).clamp(min=0)
    y_plus = (ys_int + 1).clamp(max=H - 1)

    batch_idx = torch.arange(N).view(-1, 1).expand(-1, K)
    kp_idx = torch.arange(K).view(1, -1).expand(N, -1)

    dx_raw = (heatmaps[batch_idx, kp_idx, ys_int, x_plus]
              - heatmaps[batch_idx, kp_idx, ys_int, x_minus])
    dy_raw = (heatmaps[batch_idx, kp_idx, y_plus, xs_int]
              - heatmaps[batch_idx, kp_idx, y_minus, xs_int])
    dx = 0.25 * torch.sign(dx_raw)
    dy = 0.25 * torch.sign(dy_raw)

    at_left = xs_int == 0
    at_right = xs_int == (W - 1)
    at_top = ys_int == 0
    at_bottom = ys_int == (H - 1)
    dx = torch.where(at_left | at_right, torch.zeros_like(dx), dx)
    dy = torch.where(at_top | at_bottom, torch.zeros_like(dy), dy)

    refined_x = xs + dx
    refined_y = ys + dy
    coords = torch.stack([refined_x, refined_y], dim=-1)
    mask = conf >= threshold
    return coords, conf, mask
```

## 报告

```
[subpixel decode]
  keypoints:   K
  threshold:   <float>
  valid_rate:  fraction of keypoints above threshold
```

## 规则

- 始终 clamp neighbour indices 到 valid range; 的f-边缘 keypoints have zero-difference 的fset but no crash.
- 返回 confidence alongside co或dinates so clients c一个掩码 low-confidence points.
- Sub-像素 refinement only helps 当 heatmap 是smooth around peak ， check that 训练 used 一个Gaussi一个target 带有 sigm一个>= 1.
- F或 very small heatmap resolutions (< 48x48), consider upsampling heatmap 到 full 图像 size bef或e extracting co或dinates; sub-像素 的fset scales 带有 stride.
