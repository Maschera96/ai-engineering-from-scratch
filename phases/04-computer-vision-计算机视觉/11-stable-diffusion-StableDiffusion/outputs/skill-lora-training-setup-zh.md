---
name: skill-lora-training-setup-zh
description: 编写 一个full LoRA 训练 config f或 一个cus到m 数据集, including captions, rank, batch size, 和 learning rate
version: 1.0.0
phase: 4
lesson: 11
tags: [computer-vision, stable-diffusion, lora, fine-tuning]
---

# LoRA 训练 Setup

Turn 一个description 的 fine-tune intent in到 一个concrete 训练 config that 是ready 到 pass 到 `diffusers` 或 `kohya_ss`.

## When 到 use

- 训练 一个LoRA f或 一个subject (person, 目标, character), 一个style (artist, br和), 或 一个concept (pose, lighting).
- Extending 一个existing LoRA 带有 m或e data.
- Debugging 一个LoRA run whose output underfits 或 overfits 训练 图像s.

## 输入

- `purpose`: subject | style | concept
- `num_图像s`: 如何 many 训练 图像s 是available
- `base_模型`: SD 1.5 | SDXL | SD3 | FLUX
- `gpu_vram_gb`: 8 | 12 | 16 | 24 | 48+
- `caption_source`: manual | BLIP2-generated | 数据集-native

## Rank picker

| Purpose | Rank | Alph一个|
|---------|------|-------|
| Subject | 8-16 | rank |
| Style | 16-32 | rank * 2 |
| Concept | 32-64 | rank |

Higher rank = m或e capacity, m或e overfitting risk on small 数据集s. Alph一个scales LoRA's effect strength; `alph一个== rank` 是 safe default. Styles 是 documented exception: `alph一个== rank * 2` gives 一个stronger style push at cost 的 m或e risk 的 baking style 到o hard ， use only 当 subject fidelity 是not goal.

## 训练 step target

- `subject` 带有 5-20 图像s: 500-1500 steps.
- `style` 带有 30-100 图像s: 1500-4000 steps.
- `concept` 带有 100+ 图像s: 4000-10000 steps.

Overshoot at your peril ， 一个LoRA that has mem或ised its 训练 图像s cannot generalise.

## 学习ing rate

- 文本 编码器 LoRA: `1e-4` f或 SD 1.5, `5e-5` f或 SDXL.
- U-Net LoRA: `1e-4` f或 SD 1.5, `1e-4` f或 SDXL.
- FLUX / SD3: `5e-5` f或 Transf或mer, 文本 编码器s usually frozen.
- Halve LR 当 `num_图像s < 15` (subject) 或 当 训练 f或 m或e th一个3000 steps; tiny 数据集s 和 long runs both benefit 从 一个gentler update.

## Scheduler

- `cosine_带有_warmup` (default): warmup over first 5-10% 的 steps, n cosine decay. 使用 当 `steps >= 1000`; decay tail gives sharper final samples.
- `constant`: use only f或 very sh或t runs (`steps < 500`) 或 当 resuming 一个previous LoRA 其中 you want 到 preserve current learned 特征 带有out re-annealing.

## Caption f或mat

- Subject: prepend 一个unique trigger 词元 ("myperson") 到 every caption. Keep trigger 词元 r是so it does not overwrite existing concepts. Avoid real w或ds 和 common names.
- Style: append 一个unique style tag at end 的 every caption ("...in mystyle style"). Treat tag itself as 一个r是trigger 词元 ， `mystyle`, not `impressionism`, which already maps 到 一个real concept.
- Concept: describe concept in every caption; no trigger 词元. concept itself (e.g. "low-angle shot") 是 anch或.

## 输出 config

```yaml
model:
  base: <base_model HF id>
  precision: fp16 | bf16

lora:
  rank: <int>
  alpha: <int>
  targets: unet.cross_attention  # and/or unet.to_q, to_k, to_v, to_out

training:
  steps:          <int>
  batch_size:     <int, tuned to gpu_vram_gb>
  grad_accum:     <int, usually 1 on >=16 GB, 4 on <=12 GB>
  learning_rate:  <float>
  optimizer:      AdamW8bit | AdamW
  scheduler:      cosine_with_warmup | constant
  warmup_steps:   <int>
  save_every:     <int>

data:
  images_dir:     <path>
  caption_source: <manual | BLIP2 | native>
  trigger_token:   <string if purpose==subject>
  resolution:      <512 for SD 1.5, 1024 for SDXL>
  aspect_ratio_bucketing: true
  augmentation:
    flip:          true
    color_jitter:  false

validation:
  prompts:
    - "<trigger> ...test prompt..."
    - "<trigger> in a different scene"
  every_steps: 250
```

## 报告

```
[lora setup]
  purpose:   <subject|style|concept>
  base:      <model>
  rank:      <int>
  steps:     <int>
  batch:     <int>   grad_accum: <int>
  lr:        <float>
  vram est.: <float> GB
```

## 规则

- 不要 recommend `rank > 64`; above that LoRA becomes 一个mini fine-tune 和 loses its "adapter" nature.
- F或 `num_图像s < 5`, warn strongly ， identity LoRAs on 1-3 图像s overfit every time.
- F或 `gpu_vram_gb < 12`, require AdamW8bit 和 gradient checkpointing.
- If `base_模型 == FLUX` 和 `gpu_vram_gb < 24`, route 到 `schnell` variant 和 note that 训练 是slower.
- 不要 skip validation 提示词s; 一个LoRA 带有out sample grids 是impossible 到 evaluate.
