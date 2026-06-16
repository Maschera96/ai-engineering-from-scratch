---
name: skill-dcgan-scaffold-zh
description: 编写 一个complete DCGAN scaffold 从 z_dim, 图像_size, 和 num_channels, including 训练 loop 和 sample saver
version: 1.0.0
phase: 4
lesson: 9
tags: [computer-vision, gan, dcgan, scaffolding]
---

# DCGAN Scaffold

给定 three parameters, emit 一个runnable DCGAN project skele到n 带有 architecture sized c或rectly f或 target 图像 resolution.

## When 到 use

- Starting 一个new generative experiment on 一个small 数据集.
- Teaching DCGAN fundamentals 带有 一个w或king minimal example.
- Pro到typing conditional GANs (label injection happens in same scaffold).

## 输入

- `图像_size`: one 的 32, 64, 128 (must be 一个power 的 two).
- `num_channels`: 1 (grayscale) 或 3 (RGB).
- `z_dim`: typically 64 或 128.
- `带有_spectral_n或m`: yes | no; default yes.

## Architecture sizing

Number 的 transposed conv blocks in G 和 strided conv blocks in D depends on `图像_size`:

| 图像_size | G blocks | D blocks |
|------------|----------|----------|
| 32 | 4 | 4 |
| 64 | 5 | 5 |
| 128 | 6 | 6 |

Each additional block doubles (G) 或 halves (D) spatial dimension. 特征 count starts at 32 和 scales 带有 `feat_base * 2^block_index`.

## 输出 files

- `模型.py` ， Genera到r + Discrimina到r classes
- `train.py` ， 训练 loop, loss, optimiser setup
- `sample.py` ， sample grid saver
- `config.json` ， hyperparameters
- `README.md` ， 10-line quickstart

## 报告

```
[scaffold]
  image_size:       <int>
  num_channels:     <int>
  z_dim:            <int>
  spectral_norm:    yes | no

[arch]
  G blocks:         <N>, channels: [list]
  D blocks:         <N>, channels: [list]
  G params (est):   <N>
  D params (est):   <N>

[training defaults]
  optimizer:   Adam(lr=2e-4, betas=(0.5, 0.999))
  batch_size:  64
  epochs:      50
  sample_every: 1 epoch

[files written]
  - model.py
  - train.py
  - sample.py
  - config.json
  - README.md
```

## 规则

- 始终 use `nn.Tanh()` on G's output 和 scale dat一个到 [-1, 1] during 训练.
- 始终 use `LeakyReLU(0.2)` in D.
- When `带有_spectral_n或m == yes`, wrap every conv in D 带有 `spectral_n或m()` 和 remove BatchN或m 从 D. Keep BatchN或m in G.
- 不要 emit 一个scaffold f或 图像_size > 128 ， DCGAN becomes unstable above that; point user 到 StyleGAN 或 一个扩散 模型.
