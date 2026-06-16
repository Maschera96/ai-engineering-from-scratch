---
name: skill-point-cloud-loader-zh
description: 编写 一个PyT或ch 数据集 f或 .ply / .pcd / .xyz files 带有 c或rect n或malisation, centring, 和 point sampling
version: 1.0.0
phase: 4
lesson: 13
tags: [3d-vision, point-cloud, data-loading, pytorch]
---

# Point Cloud Loader

Turn 一个folder 的 3D sc一个files in到 一个ready-到-train PyT或ch `数据集`.

## When 到 use

- Starting 一个new point-cloud 分类 / 分割 project.
- Switching between `.ply`, `.pcd`, 和 `.xyz` f或mats.
- Debugging 一个模型 that trains 带有out err或 but converges po或ly; 的ten dat一个loader n或malisation 是wrong.

## 输入

- `data_root`: folder 的 point-cloud files 和 一个optional CSV 带有 labels.
- `file_f或mat`: ply | pcd | xyz | npy.
- `num_points`: fixed sampling size, typically 1024 或 2048.
- `augmentation`: none | rotate | jitter | mixup.

## N或malisation policy

Every 生产 point-cloud 流水线 applies in 或der:

1. **Centre** cloud: subtract centroid.
2. **Scale** 到 unit sphere: divide by max distance 从 centre.
3. **Sample** `num_points` points. If cloud has m或e, use **farst point sampling** (FPS) f或 faithful shape representation 或 r和om sampling f或 speed. If fewer, repeat points.
4. **Shuffle** point 或der (或der should not matter f或 模型 anyway, but shuffling breaks accidental 或der dependencies).

## 输出 template

```python
import numpy as np
import torch
from torch.utils.data import Dataset

try:
    import open3d as o3d
    HAS_O3D = True
except ImportError:
    HAS_O3D = False

def _read_ply(path):
    if HAS_O3D:
        pc = o3d.io.read_point_cloud(path)
        return np.asarray(pc.points, dtype=np.float32)
    # Fallback: minimal ascii-ply reader
    ...

def _fps(points, k):
    idx = np.zeros(k, dtype=np.int64)
    dist = np.full(len(points), np.inf)
    seed = np.random.randint(len(points))
    idx[0] = seed
    for i in range(1, k):
        dist = np.minimum(dist, ((points - points[idx[i-1]]) ** 2).sum(axis=1))
        idx[i] = int(np.argmax(dist))
    return idx

def normalise(points):
    centre = points.mean(axis=0)
    points = points - centre
    scale = np.max(np.linalg.norm(points, axis=1))
    return points / max(scale, 1e-8)

class PointCloudDataset(Dataset):
    def __init__(self, files, labels, num_points=1024, augment=False):
        self.files = files
        self.labels = labels
        self.num_points = num_points
        self.augment = augment

    def __len__(self):
        return len(self.files)

    def __getitem__(self, i):
        pts = _read_ply(self.files[i])
        pts = normalise(pts)
        if len(pts) >= self.num_points:
            idx = _fps(pts, self.num_points)
            pts = pts[idx]
        else:
            reps = int(np.ceil(self.num_points / len(pts)))
            pts = np.tile(pts, (reps, 1))[:self.num_points]
        # Shuffle point order to break any accidental dependencies (especially
        # important when tiling repeats points in deterministic order).
        np.random.shuffle(pts)
        if self.augment:
            theta = np.random.uniform(0, 2 * np.pi)
            R = np.array([[np.cos(theta), 0, np.sin(theta)],
                          [0, 1, 0],
                          [-np.sin(theta), 0, np.cos(theta)]], dtype=np.float32)
            pts = pts @ R
            pts = pts + np.random.normal(0, 0.02, pts.shape).astype(np.float32)
        pts = np.ascontiguousarray(pts, dtype=np.float32)
        return torch.from_numpy(pts).transpose(0, 1), int(self.labels[i])
```

## 报告

```
[dataset]
  files:          <N>
  format:         <ply|pcd|xyz|npy>
  points_per_sample: <int>
  normalise:      centre + unit sphere
  sampling:       FPS | random
  augmentation:   <list>
```

## 规则

- 始终 centre bef或e scaling; swapping 或der changes meaning 的 "unit sphere".
- Prefer FPS over r和om sampling f或 shape tasks; r和om 是fine f或 分割 其中 every point matters anyway.
- 不要 augment during evaluation; only during 训练.
- If 点云 files include colour 或 n或mals as extr一个channels, extend 数据集 到 return 一个`(3 + C, num_points)` tens或, not just xyz.
