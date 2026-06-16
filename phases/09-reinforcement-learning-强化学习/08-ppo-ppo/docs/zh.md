# Proximal 策略 优化 (PPO)

> A2C throws away each  rollout after one update. PPO wraps the 策略梯度 in a clipped importance 比例 so you can do 10+ epochs on the same 数据 without the 策略 exploding. Schulman et al. (2017). Still the default policy-gradient algorithm in 2026.

**类型：** Build
**语言：** Python
**先修：** Phase 9 · 06 (REINFORCE), Phase 9 · 07 (Actor-Critic)
**时间：** 约 75 分钟

## 问题

A2C (Lesson 07) is on-policy: the 梯度 `E_{π_θ}[A · ∇ log π_θ]` requires 数据 sampled from the *current* `π_θ`. Take one update, and `π_θ` changes; the 数据 you used is now off-policy. Re-use it and your 梯度 is biased.

Rollouts are expensive. On Atari, one  rollout across 8 envs × 128 步骤 = 1024 transitions and a dozen seconds of 环境 time. Throwing that away after one 梯度 步骤 is wasteful.

Trust Region 策略 优化 (TRPO, Schulman 2015) was the first fix: constrain each update so the KL divergence between old and new 策略 stays below `δ`. Theoretically clean, but requires a conjugate-gradient solve per update. Nobody runs TRPO in 2026.

PPO (Schulman et al. 2017) replaces the hard trust-region constraint with a simple clipped 目标. One extra line of code. Ten epochs per  rollout. No conjugate gradients. Good-enough theoretical guarantees. Nine years later it is still the default policy-gradient algorithm for everything from MuJoCo to RLHF.

## 概念

![PPO clipped surrogate objective: ratio clipping at 1 ± ε](../assets/ppo.svg)

**The importance 比例.**

`r_t(θ) = π_θ(a_t | s_t) / π_{θ_old}(a_t | s_t)`

这is the 似然 比例 of the new 策略 vs the 策略 that collected the 数据. `r_t = 1` means no change. `r_t = 2` means the new 策略 is twice as likely to take `a_t` as the old.

**The clipped surrogate.**

`L^{CLIP}(θ) = E_t [ min( r_t(θ) A_t, clip(r_t(θ), 1-ε, 1+ε) A_t ) ]`

Two terms:

- 如果the advantage `A_t > 0` and the 比例 tries to grow past `1 + ε`, the clip flattens the 梯度 — don't push a good 动作 further than `+ε` above old 概率.
- 如果the advantage `A_t < 0` and the 比例 tries to grow past `1 - ε` (meaning we would make a bad 动作 more likely compared to its clipped reduction), the clip caps the 梯度 — don't push a bad 动作 below `-ε`.

这个`min` handles the other direction: if the 比例 has moved in the *beneficial* direction, you still get the 梯度 (no clipping on the side that would hurt you).

Typical `ε = 0.2`. Plot the 目标 as a 函数 of `r_t`: a piecewise-linear 函数 with a flat roof on the "good side" and a flat floor on the "bad side."

**The full PPO 损失.**

`L(θ, φ) = L^{CLIP}(θ) - c_v · (V_φ(s_t) - V_t^{target})² + c_e · H(π_θ(·|s_t))`

Same actor-critic structure as A2C. Three coefficients, usually `c_v = 0.5`, `c_e = 0.01`, `ε = 0.2`.

**The 训练 循环.**

1. Collect `N × T` transitions across `N` 并行 envs for `T` 步骤 each.
2. 计算 advantages (GAE), freeze them as constants.
3. Freeze `π_{θ_old}` as a snapshot of current `π_θ`.
4. For `K` epochs, for each minibatch of `(s, a, A, V_target, log π_old(a|s))`:
   - 计算 `r_t(θ) = exp(log π_θ(a|s) - log π_old(a|s))`.
   - Apply `L^{CLIP}` + value 损失 + 熵.
   - 梯度 步骤.
5. Discard the  rollout. Return to 步骤 1.

`K = 10` and minibatches of 64 is a standard hyperparameter set. PPO is robust: the exact numbers rarely matter within ±50%.

**KL-penalty variant.** The original paper proposed an 替代方案 using an adaptive KL penalty: `L = L^{PG} - β · KL(π_θ || π_old)` with `β` adjusted based on observed KL. The clipping version became dominant; the KL variant survives in RLHF (where KL to the 参考 策略 is a separate constraint you always want anyway).

## 动手构建

### 步骤 1: capture `log π_old(a | s)` at  rollout time

```python
for step in range(T):
    probs = softmax(logits(theta, state_features(s)))
    a = sample(probs, rng)
    s_next, r, done = env.step(s, a)
    buffer.append({
        "s": s, "a": a, "r": r, "done": done,
        "v_old": value(w, state_features(s)),
        "log_pi_old": log(probs[a] + 1e-12),
    })
    s = s_next
```

这个snapshot is taken once, at  rollout time. It does not change during the update epochs.

### 步骤 2: 计算 GAE advantages (Lesson 07)

Same as A2C. Normalize across the 批次.

### 步骤 3: clipped surrogate update

```python
for _ in range(K_EPOCHS):
    for mb in minibatches(buffer, size=64):
        for rec in mb:
            x = state_features(rec["s"])
            probs = softmax(logits(theta, x))
            logp = log(probs[rec["a"]] + 1e-12)
            ratio = exp(logp - rec["log_pi_old"])
            adv = rec["advantage"]
            surrogate = min(
                ratio * adv,
                clamp(ratio, 1 - EPS, 1 + EPS) * adv,
            )
            # backprop -surrogate, add value loss, subtract entropy
            grad_logpi = onehot(rec["a"]) - probs
            if (adv > 0 and ratio >= 1 + EPS) or (adv < 0 and ratio <= 1 - EPS):
                pg_grad = 0.0  # clipped
            else:
                pg_grad = ratio * adv
            for i in range(N_ACTIONS):
                for j in range(N_FEAT):
                    theta[i][j] += LR * pg_grad * grad_logpi[i] * x[j]
```

这个"clipped → zero 梯度" pattern is the heart of PPO. If the new 策略 has already drifted too far in the beneficial direction, the update stops.

### 步骤 4: value and 熵

Add standard MSE to the critic 目标 and an 熵 bonus on the actor, same as A2C.

### 步骤 5: diagnostics

Three things to watch every update:

- **Mean KL** `E[log π_old - log π_θ]`. Should stay in `[0, 0.02]`. If it blows past `0.1`, reduce `K_EPOCHS` or `LR`.
- **Clip fraction** — the fraction of 样本 whose 比例 lies outside `[1-ε, 1+ε]`. Should be `~0.1-0.3`. If `~0`, the clip never triggers → raise `LR` or `K_EPOCHS`. If `~0.5+`, you are over-fitting the  rollout → lower them.
- **Explained 方差** `1 - Var(V_target - V_pred) / Var(V_target)`. Critic 质量 指标. Should climb toward 1 as the critic learns.

## Pitfalls

- **Clip coefficient mistuned.** `ε = 0.2` is the de-facto standard. Going to `0.1` makes updates too timid; `0.3+` invites instability.
- **Too many epochs.** `K > 20` routinely destabilizes because the 策略 drifts far from `π_old`. Cap epochs, especially for large networks.
- **No 奖励 归一化.** Large 奖励 scales eat into the clip range. Normalize rewards (running std) before computing advantages.
- **Forgetting advantage 归一化.** Per-batch zero-mean/unit-std 归一化 is standard. Skipping it wrecks PPO on most benchmarks.
- **学习 速率 not decayed.** PPO benefits from linear LR 衰减 to zero. Constant LR is often worse.
- **Importance 比例 math 错误.** Always `exp(log_new - log_old)` for numerical stability, not `new / old`.
- **Wrong 梯度 sign.** Maximize the surrogate = *minimize* `-L^{CLIP}`. A flipped sign is the most common PPO bug.

## 实际使用

PPO is 2026's default RL algorithm across a surprising number of domains:

|Use case|PPO variant|
|----------|-------------|
|MuJoCo / robotics control|PPO with Gaussian 策略, GAE(0.95)|
|Atari / discrete games|PPO with categorical 策略, rolling 128-步骤 rollouts|
|RLHF for LLMs|PPO with KL penalty to 参考 模型, 奖励 from RM at end of 响应|
|Large-scale game agents|IMPALA + PPO (AlphaStar, OpenAI Five)|
|推理 LLMs|GRPO (Lesson 12) — PPO variant without critic|
|Preference-only 数据|DPO — closed-form collapsing of PPO+KL, no online 采样|

这个PPO *损失 shape* — clipped surrogate + value + 熵 — is the scaffolding for DPO, GRPO, and nearly every RLHF 流水线.

## 交付成果

Save as `outputs/skill-ppo-trainer.md`:

```markdown
---
name: ppo-trainer
description: Produce a PPO training config and a diagnostic plan for a given environment.
version: 1.0.0
phase: 9
lesson: 8
tags: [rl, ppo, policy-gradient]
---

Given an environment and training budget, output:

1. Rollout size. `N` envs × `T` steps.
2. Update schedule. `K` epochs, minibatch size, LR schedule.
3. Surrogate params. `ε` (clip), `c_v`, `c_e`, advantage normalization on.
4. Advantage. GAE(`λ`) with explicit `γ` and `λ`.
5. Diagnostics plan. KL, clip fraction, explained variance thresholds with alerts.

Refuse `K > 30` or `ε > 0.3` (unsafe trust region). Refuse any PPO run without advantage normalization or KL/clip monitoring. Flag clip fraction sustained above 0.4 as drift.
```

## 练习

1. **Easy.** Run PPO on 4×4 GridWorld with `ε=0.2, K=4`. Compare 样本 efficiency to A2C (one 轮次 per  rollout) at matched env 步骤.
2. **Medium.** Sweep `K ∈ {1, 4, 10, 30}`. Plot return vs env 步骤 and track mean KL per update. At what `K` does KL explode on this 任务?
3. **Hard.** Replace the clipped surrogate with an adaptive KL penalty (`β` doubled if `KL > 2·target`, halved if `KL < target/2`). Compare final return, stability, and clip-free-ness.

## Key Terms

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|Importance 比例|"r_t(θ)"|`π_θ(a\|s) / π_old(a\|s)`; deviation from the 策略 that collected the 数据.|
|Clipped surrogate|"PPO's main trick"|`min(r·A, clip(r, 1-ε, 1+ε)·A)`; flat 梯度 past the clip on beneficial side.|
|Trust region|"TRPO / PPO intent"|限制 each update's KL to guarantee monotone improvement.|
|KL penalty|"Soft trust region"|替代方案 PPO: `L - β · KL(π_θ \|\|π_old)`. Adaptive `β`.|
|Clip fraction|"How often clipping triggers"|Diagnostic — should be 0.1-0.3; outside means mistuned.|
|Multi-epoch 训练|"数据 reuse"|K epochs on each  rollout; 方差 成本 traded for 样本 efficiency.|
|On-policy-ish|"Mostly on-policy"|PPO is nominally on-policy but K>1 epochs uses slightly-off-policy 数据 safely.|
|PPO-KL|"The other PPO"|KL-penalty variant; used in RLHF where KL-to-reference is already a constraint.|

## 延伸阅读

- [Schulman et al. (2017). Proximal Policy Optimization Algorithms](https://arxiv.org/abs/1707.06347) — the paper.
- [Schulman et al. (2015). Trust Region Policy Optimization](https://arxiv.org/abs/1502.05477) — TRPO, PPO's predecessor.
- [Andrychowicz et al. (2021). What Matters In On-Policy RL? A Large-Scale Empirical Study](https://arxiv.org/abs/2006.05990) — every PPO hyperparameter ablated.
- [Ouyang et al. (2022). Training language models to follow instructions with human feedback](https://arxiv.org/abs/2203.02155) — InstructGPT; the PPO-in-RLHF recipe.
- [OpenAI Spinning Up — PPO](https://spinningup.openai.com/en/latest/algorithms/ppo.html) — clean modern exposition with PyTorch.
- [CleanRL PPO implementation](https://github.com/vwxyzjn/cleanrl) — 参考 single-file PPO used by many papers.
- [Hugging Face TRL — PPOTrainer](https://huggingface.co/docs/trl/main/en/ppo_trainer) — the 生产 recipe for PPO on 语言模型s; read alongside Lesson 09 (RLHF).
- [Engstrom et al. (2020). Implementation Matters in Deep Policy Gradients](https://arxiv.org/abs/2005.12729) — the "37 code-level optimizations" paper; which PPO tricks are load-bearing and which are folklore.
