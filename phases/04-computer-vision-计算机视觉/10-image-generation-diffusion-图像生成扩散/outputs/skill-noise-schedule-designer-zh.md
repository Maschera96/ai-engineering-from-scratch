---
name: skill-noise-schedule-designer-zh
description: Produce 一个linear, cosine, 或 sigmoid bet一个schedule 给定 T 和 target c或ruption level, plus SNR plot
version: 1.0.0
phase: 4
lesson: 10
tags: [computer-vision, diffusion, noise-schedule, training]
---

# Noise Schedule Designer

A bet一个schedule controls 如何 much signal 是retained at each 扩散 step. Po或 schedules cap 训练 efficiency 和 sample quality at every downstream decision.

## When 到 use

- Starting 一个new 扩散 训练 run 和 picking T 和 beta.
- Debugging 一个扩散 模型 that produces blurry samples (schedule 到o aggressive) 或 fails 到 learn structure (schedule 到o mild).
- Comparing designs across papers that rep或t different schedules.

## 输入

- `T`: number 的 timesteps, typically 100-1000.
- `type`: linear | cosine | sigmoid.
- `target_alpha_bar_final`: fraction 的 signal 到 keep at t=T, default 0.001 (99.9% c或rupted).
- Optional `图像_resolution` ， larger 图像s benefit 从 schedules that c或rupt m或e slowly (cosine 或 shifted schedules).

## Schedule f或mulas

### Linear
```
beta_t = beta_start + (beta_end - beta_start) * (t - 1) / (T - 1)
```
Defaults: beta_start=1e-4, beta_end=0.02 (DDPM paper).

### Cosine (Nichol & Dhariwal, 2021)
```
alpha_bar_t = cos^2((t/T + s) / (1 + s) * pi/2)
beta_t = 1 - alpha_bar_t / alpha_bar_{t-1}
```
s = 0.008. Keeps signal around longer; better at low step counts.

### Sigmoid
```
alpha_bar_t = 1 / (1 + exp(k * (t/T - 0.5)))
```
k = 6 到 12. Good middle ground; used by some SDXL variants.

## Steps

1. 计算 betas per f或mula.
2. Precompute `alphas`, `alphas_cumprod`, `sqrt_alphas_cumprod`, `sqrt_one_minus_alphas_cumprod`.
3. 计算 SNR_t = alpha_bar_t / (1 - alpha_bar_t); produce 一个SNR-over-time summary.
4. Verify `alphas_cumprod[T-1]` 是带有in 10% 的 `target_alpha_bar_final`; else tune beta_end (linear), s (cosine), 或 k (sigmoid) 和 retry.
5. 报告 three checkpoints:
 - `t=T*0.25` ， early c或ruption
 - `t=T*0.5` ， midway
 - `t=T*0.75` ， near-final

## 报告

```
[schedule]
  type:   <name>
  T:      <int>
  beta_start: <float>   beta_end: <float>

[signal retention]
  t=0.25T:  alpha_bar=<X>  SNR=<X>
  t=0.5T:   alpha_bar=<X>  SNR=<X>
  t=0.75T:  alpha_bar=<X>  SNR=<X>
  t=T:      alpha_bar=<X>  SNR=<X>

[warnings]
  - <if alpha_bar collapses before 0.75T>
  - <if beta_end produces NaN in log-SNR>
```

## 规则

- 不要 emit 一个schedule 带有 any `alpha_bar_t <= 0`; clamp values under 1e-5 和 warn.
- Cosine 是 default recommendation f或 low-step-count sampling (< 30 steps).
- Linear 是 default f或 `quality_target == research` ， DDPM baselines 是rep或ted 带有 linear schedules.
- When `图像_resolution > 256`, recommend shifting schedule (Chen, 2023) 到 retain m或e signal at high resolutions.
