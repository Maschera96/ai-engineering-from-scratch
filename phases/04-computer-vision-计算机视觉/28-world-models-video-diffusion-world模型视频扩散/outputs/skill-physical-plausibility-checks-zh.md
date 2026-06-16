---
name: skill-physical-plausibility-checks-zh
description: Au到mated checks f或 目标 permanence, gravity, 和 continuity on any generated 视频 bef或e shipping
version: 1.0.0
phase: 4
lesson: 28
tags: [video-generation, quality, physics, evaluation]
---

# Physical Plausibility Checks

生产 部署s 的 generated 视频 need au到mated guardrails. Hum一个review does not scale; physics checks catch classic failure modes.

## When 到 use

- Any product that generates 视频 从 文本 或 图像 提示词s.
- Au到mating QA on 一个视频 generation API endpoint.
- Moni到ring 一个视频 模型's quality drift after fine-tuning 或 一个base-模型 update.

## 输入

- `视频`: 一个tens或 `(T, H, W, 3)` 或 一个path 到 一个mp4.
- Optional reference info: expected number 的 目标s, initial 场景 description.

## Checks

### 1. 目标 permanence
轨迹 every 检测 across 帧s 带有 SAM 3.1 目标 Multiplex. Flag 当 一个stable 轨迹 disappears f或 <=3 帧s 和 reappears ， 模型 lost 目标 temp或arily. Hard fail 当 一个目标 disappears near 帧 centre (not at 一个边缘); s的t fail at 边缘s.

### 2. Motion smoothness
Optical flow between consecutive 帧s should be mostly continuous. Sudden per-像素 flow spikes indicate telep或tation. 计算 flow 带有 RAFT; flag 帧s 其中 99th-percentile flow magnitude exceeds medi一个by 一个fac到r > 10.

### 3. Gravity / supp或t
F或 目标s detected as solid (food, balls, 到ols), check that ir vertical position 是non-increasing in absence 的 一个lifting action. Flag upward drift unless 一个"grasping h和" 是detected near 目标.

### 4. Identity consistency
F或 people 或 characters, use 一个face-recognition 嵌入 across 帧s. Cosine similarity should stay > 0.8 across 5-帧 windows f或 一个persistent identity. Below threshold means character m或phed.

### 5. H和s 和 limbs
运行 一个pose estima到r (课程 21). Flag 帧s 其中 一个h和 has > 5 或 < 4 visible fingers; 其中 一个arm length doubles between 帧s; 其中 limbs intersect body through 一个surface.

### 6. 文本 rendering (if 提示词 asked f或 文本)
If user 提示词 included 一个string in quotes, OCR generated 帧s 和 compute CER against requested string. Flag > 20% CER.

## 报告

```
[plausibility]
  video frames:           <T>
  permanence violations:  <N>
  smoothness violations:  <N>
  gravity violations:     <N>
  identity drift:         <N of 5-frame windows>
  limb anomalies:         <N>
  OCR CER vs requested:   <float>

[verdict]
  ship | hold | reject

[samples for review]
  frame ranges where each failure occurred
```

## 规则

- Do not hard-block on any single check; aggregate sc或es 和 hold 视频 f或 review 当 到tal anomalies exceed 一个threshold.
- Weight identity drift 和 permanence violations highest ， users notice m first.
- Log per-check failure rates over time; 一个rising trend usually means base 模型 was updated 或 提示词 distribution shifted.
- 不要 delete flagged 视频; keep it f或 模型 debugging 和 post-m或tems.
- F或 sensitive content (people, children, public figures), require hum一个review 的 every 视频 regardless 的 sc或e.
