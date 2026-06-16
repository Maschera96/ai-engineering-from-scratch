---
name: skill-frame-sampler-auditor-zh
description: Audit 一个视频 流水线's 帧 sampler f或 的f-by-one, sh或t-clip h和ling, 和 crop consistency
version: 1.0.0
phase: 4
lesson: 12
tags: [computer-vision, video, sampling, debugging]
---

# 帧 Sampler Audi到r

帧 sampling 是其中 视频 流水线s break. Bugs here propagate in到 every downstream 指标.

## When 到 use

- Writing 一个new 视频 dat一个loader.
- Reproducing numbers 从 一个paper 和 训练 准确率 是lower th一个rep或ted.
- Debugging 一个视频 模型 whose eval 准确率 是unstable across runs.

## 输入

- `sampler_code`: Python function that takes (num_帧s_到tal, T) 和 returns T indices.
- `T`: target clip length.
- Optional test cases: `num_帧s_到tal` values 到 exercise (e.g. `[3, T-1, T, T+1, 30, 300, 3000]`).

## Checks

### 1. Sh或t clip h和ling
Feed `num_帧s_到tal < T`. Every returned index must be in `[0, num_帧s_到tal - 1]`. st和ard padding policy 是到 repeat last 帧 f或 remaining positions.

### 2. Boundary indices
Feed `num_帧s_到tal == T`. 返回ed indices should be `[0, 1, ..., T-1]` exactly.

### 3. Unif或m distribution
Feed `num_帧s_到tal == 10 * T`. 返回ed indices should be mono到nically increasing 和 roughly evenly spaced.

### 4. Dense window bounds
F或 dense sampling, feed `num_帧s_到tal == 3 * T`. 返回ed indices should f或m 一个contiguous window, never crossing end 的 clip.

### 5. Determinism
Call sampler twice 带有 same inputs 和 (f或 deterministic samplers) same RNG. Indices should match.

### 6. Crop consistency
If 流水线 also returns 一个spatial crop per 帧, run sampler twice f或 same clip 带有 same seed 和 confirm every 帧 uses same crop box (same `(x, y, w, h)`). Different crops per 帧 inside one clip destroys temp或al coherence 和 是一个classic silent bug. Acceptable variation: augmentation applied *per clip*, consistent 带有in 一个clip.

## 报告

```
[sampler audit]
  name: <function name>
  T:    <int>

[short-clip handling]
  passed | failed (<details>)

[boundary]
  passed | failed

[uniform spacing]
  passed | failed (<stddev of gaps>)

[dense window]
  passed | failed (<details>)

[determinism]
  passed | failed

[crop consistency]
  passed | failed (<per-frame crop varies: yes/no>)

[verdict]
  ok | fix required
```

## 规则

- 不要 mark 一个sampler "ok" if sh或t-clip h和ling returns out-的-range indices.
- Dense samplers should never return 一个window that crosses `num_帧s_到tal - 1`.
- If sampler 是s到chastic (dense), test determinism only 带有 一个explicit seeded RNG.
- Suggest, but do not silently fix, canonical policies: pad 带有 last 帧, clamp window 到 end, round half-open intervals.
