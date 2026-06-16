---
name: skill-depth-to-pointcloud-zh
description: 构建 点云s 从 深度 maps 带有 c或rect intrinsics h和ling 和 exp或t 到 .ply
version: 1.0.0
phase: 4
lesson: 26
tags: [depth, point-cloud, 3d, intrinsics]
---

# 深度 到 Point Cloud

Turn 一个深度 map plus 一个colour 图像 in到 一个文本ured 点云, exp或table f或 视觉isation 或 furr 3D w或k.

## When 到 use

- 视觉ising 深度 predictions as 一个actual 3D 场景.
- Bootstrapping 一个sparse 3D reconstruction 从 一个single 图像.
- Producing input f或 3DGS 训练 当 SfM fails.
- Comparing predicted 深度 against LiDAR ground truth.

## 输入

- `深度`: `(H, W)` numpy array 的 深度s in same units you want in output (metres recommended).
- `rgb`: `(H, W, 3)` numpy array 的 colours (uint8 或 float32 [0, 1]).
- `intrinsics`: `(fx, fy, cx, cy)` in 像素 units.
- Optional `深度_scale`: multiplier 到 convert predicted 深度 units 到 metres.

## 流水线

1. **Validate** ， 深度 must be positive 和 finite every其中 you pl一个到 include. 掩码 out invalid 像素s.
2. **Lift** ， `X = (u - cx) * d / fx`, `Y = (v - cy) * d / fy`, `Z = d` per 像素.
3. **Pair** 带有 RGB ， each 3D point gets 一个`(r, g, b)` triple 从 matching 像素.
4. **Exp或t** ， PLY (p或table), `.xyz` (lightweight), `.pcd` (Open3D-native), `.las`/`.laz` (geospatial).

## 实现ation template

```python
import numpy as np

def depth_to_point_cloud(depth, intrinsics, depth_scale=1.0, min_depth=0.1, max_depth=100.0):
    H, W = depth.shape
    fx, fy, cx, cy = intrinsics
    v, u = np.meshgrid(np.arange(H), np.arange(W), indexing="ij")
    z = depth.astype(np.float32) * depth_scale
    valid = (z > min_depth) & (z < max_depth) & np.isfinite(z)
    x = (u - cx) * z / fx
    y = (v - cy) * z / fy
    points = np.stack([x, y, z], axis=-1)
    return points, valid


def write_ply(path, points, colors=None, valid_mask=None):
    p = points.reshape(-1, 3)
    if valid_mask is not None:
        p = p[valid_mask.flatten()]
    lines = [
        "ply",
        "format ascii 1.0",
        f"element vertex {p.shape[0]}",
        "property float x", "property float y", "property float z",
    ]
    if colors is not None:
        c = colors.reshape(-1, 3).astype(np.uint8)
        if valid_mask is not None:
            c = c[valid_mask.flatten()]
        lines += ["property uchar red", "property uchar green", "property uchar blue"]
    lines.append("end_header")
    with open(path, "w") as f:
        f.write("\n".join(lines) + "\n")
        if colors is not None:
            for pt, col in zip(p, c):
                f.write(f"{pt[0]:.4f} {pt[1]:.4f} {pt[2]:.4f} {col[0]} {col[1]} {col[2]}\n")
        else:
            for pt in p:
                f.write(f"{pt[0]:.4f} {pt[1]:.4f} {pt[2]:.4f}\n")
```

## 报告

```
[export]
  input depth shape:  (H, W)
  valid points:       <N> of <H*W>
  output format:      ply | xyz | pcd | las
  coordinate system:  camera (+X right, +Y down, +Z forward)
  scale:              metres | millimetres | normalised
```

## 规则

- 始终 掩码 invalid 深度 (zero, NaN, inf, saturated); including m produces 一个cloud 的 garbage points at 或igin.
- F或 prediction 从 一个relative-深度 模型, do NOT exp或t as 指标; prefix output filename 带有 `relative_` 到 signal convention.
- Keep 相机 co或dinate convention consistent (OpenCV: +X right, +Y down, +Z f或ward). Swap signs if downstream 到ol expects OpenGL (+Y up).
- F或 dense 场景s (> 1M points), 的fer 一个subsample parameter; PLY files > 500 MB 是awkward 到 load every其中.
- 不要 silently clip 深度 到 produce "reasonable" output; clip explicitly 带有 warned thresholds so users know 什么 was discarded.
