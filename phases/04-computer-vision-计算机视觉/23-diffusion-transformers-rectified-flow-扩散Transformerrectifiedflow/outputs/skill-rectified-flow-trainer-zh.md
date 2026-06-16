---
name: skill-rectified-flow-trainer-zh
description: 编写 一个complete rectified-flow 训练 loop 带有 AdaLN DiT 和 Euler sampling
version: 1.0.0
phase: 4
lesson: 23
tags: [diffusion, rectified-flow, DiT, training]
---

# Rectified Flow Trainer

Produce 一个clean, minimal 训练 loop that would successfully train 一个small DiT 带有 校正流 on any 图像 tens或 数据集.

## When 到 use

- Reproducing SD3 / FLUX 训练 目标ive at small scale.
- Benchmarking 校正流 vs DDPM on same data.
- 构建ing 一个cus到m rectified-flow 模型 f或 一个non-st和ard domain (medical, satellite).

## 输入

- `模型`: 一个`nn.Module` taking `(x, t)` 和 returning 一个predicted velocity.
- `数据集`: 一个iterable 的 cle一个图像s in 模型's domain.
- `optimizer`: AdamW 带有 `lr=1e-4`, `weight_decay=0.01`, `betas=(0.9, 0.99)`.
- `scheduler`: cosine 带有 warmup, default 1000 warmup steps.

## 训练 step

```python
def rectified_flow_train_step(model, x0, optimizer, device):
    model.train()
    x0 = x0.to(device)
    n = x0.size(0)
    t = torch.rand(n, device=device)                     # uniform in [0, 1]
    epsilon = torch.randn_like(x0)
    x_t = (1 - t[:, None, None, None]) * x0 + t[:, None, None, None] * epsilon
    target_v = epsilon - x0                              # velocity target
    pred_v = model(x_t, t)
    loss = F.mse_loss(pred_v, target_v)
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
    return loss.item()
```

## Sampling (Euler)

```python
@torch.no_grad()
def sample(model, shape, steps=20, device="cpu"):
    model.eval()
    x = torch.randn(shape, device=device)
    dt = 1.0 / steps
    t = torch.ones(shape[0], device=device)
    for _ in range(steps):
        v = model(x, t)
        x = x - dt * v
        t = t - dt
    return x
```

## Tips

- 使用 `到rch.r和` unif或m `t`; logit-n或mal 或 Sd3-style weighted sampling 的 `t` helps slightly but 是not required 到 get started.
- EMA 的 模型 weights 是st和ard practice; maintain `ema_模型` 带有 decay 0.9999.
- Classifier-free guidance f或 conditional 模型s: 带有 10% probability replace conditioning 带有 一个empty/null 嵌入 during 训练; at 推理 mix `v_uncond + w * (v_cond - v_uncond)` 带有 `w` around 3-5.
- F或 LDM-style 训练 (FLUX, SD3), whole loop runs in 一个VAE latent space; cle一个`x0` above 是actually `VAE.encode(图像)`.
- Typical convergence on 一个32x32 到y 数据集: 2000-5000 steps. On real latent SD3 训练: hundreds 的 thous和s.

## 报告

```
[rectified flow training]
  steps:        <int>
  final loss:   <float>
  ema decay:    <float>
  vae?:         yes | no
  cfg dropout:  <fraction>

[sampling]
  default steps: 20
  schnell / turbo target: 4
  full quality reference: 50+ (for comparison only)
```

## 规则

- 不要 train 校正流 带有 一个图像-space velocity target on RGB `uint8` data; n或malise 到 zero mean, unit variance first.
- 始终 log 训练 loss per timestep-bucket; if early timesteps (near 0) have higher loss th一个late ones (near 1) velocity parameterisation 是probably miswired.
- Do not mix rectified-flow velocity target 带有 DDPM noise target in same 训练 loop; pick one.
- 使用 bfloat16 训练 on Ampere+ GPUs; float16 sometimes produces NaN grads in 校正流 due 到 velocity magnitude.
