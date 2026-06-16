# Actor-Critic — A2C and A3C

> REINFORCE is noisy. Add a critic that learns `V̂(s)`, subtract it from the return, and you get an advantage that has the same expectation but far lower 方差. That is actor-critic. A2C runs it synchronously; A3C runs it across threads. Both are the mental 模型 for every modern deep-RL method.

**类型：** Build
**语言：** Python
**先修：** Phase 9 · 04 (TD 学习), Phase 9 · 06 (REINFORCE)
**时间：** 约 75 分钟

## 问题

Vanilla REINFORCE works, but its 方差 is terrible. Monte Carlo returns `G_t` can swing over a factor of 10 between episodes. Multiplying that 噪声 by `∇ log π` and averaging produces a 梯度 estimator that takes thousands of episodes to move the 策略 the same distance you could move it with far fewer DQN updates.

这个方差 comes from using raw returns. If you subtract a 基线 `b(s_t)` — any 函数 of 状态, including a learned value — the expectation is unchanged and the 方差 drops. The best tractable 基线 is `V̂(s_t)`. Now the quantity multiplying `∇ log π` is the *advantage*:

`A(s, a) = G - V̂(s)`

一个动作 is good if it produced above-average return; bad if below. REINFORCE with a learned critic is *actor-critic*. The critic gives the actor a low-variance teacher. This is every deep-policy method after 2015 (A2C, A3C, PPO, SAC, IMPALA).

## 概念

![Actor-critic: policy net plus value net, TD residual as advantage](../assets/actor-critic.svg)

**Two networks, one shared 损失:**

- **Actor** `π_θ(a | s)`: the 策略. Sampled to act. 训练后的 with 策略梯度.
- **Critic** `V_φ(s)`: estimates expected return from 状态. 训练后的 to minimize `(V_φ(s) - target)²`.

**The advantage.** Two standard forms:

- *MC advantage:* `A_t = G_t - V_φ(s_t)`. Unbiased, higher 方差.
- *TD advantage:* `A_t = r_{t+1} + γ V_φ(s_{t+1}) - V_φ(s_t)`. Biased (uses `V_φ`), far lower 方差. Also called the *TD residual* `δ_t`.

**n-step advantage.** Interpolate between the two:

`A_t^{(n)} = r_{t+1} + γ r_{t+2} + … + γ^{n-1} r_{t+n} + γ^n V_φ(s_{t+n}) - V_φ(s_t)`

`n = 1` is pure TD. `n = ∞` is MC. Most implementations use `n = 5` for Atari, `n = 2048` for PPO on MuJoCo.

**Generalized Advantage Estimation (GAE).** Schulman et al. (2016) proposed an exponentially weighted average over all n-step advantages:

`A_t^{GAE} = Σ_{l=0}^{∞} (γλ)^l δ_{t+l}`

with `λ ∈ [0, 1]`. `λ = 0` is TD (low 方差, high 偏差). `λ = 1` is MC (high 方差, unbiased). `λ = 0.95` is the 2026 default — tune until the 偏差/方差 dial is where you want it.

**A2C: synchronous advantage actor-critic.** Collect `T` 步骤 across `N` 并行 environments. 计算 advantages for each 步骤. Update actor and critic on the combined 批次. Repeat. The simpler, more-scalable sibling of A3C.

**A3C: 异步 advantage actor-critic.** Mnih et al. (2016). Spawn `N` worker threads, each running an env. Each worker computes gradients locally on its own  rollout, then asynchronously applies them to a shared 参数 服务器. No replay buffer needed — workers decorrelate by running different trajectories. A3C proved you could 训练 on CPUs at 规模. In 2026, GPU-based A2C (batched 并行 envs) dominates because GPUs want large batches.

**The combined 损失.**

`L(θ, φ) = -E[ A_t · log π_θ(a_t | s_t) ]  +  c_v · E[(V_φ(s_t) - G_t)²]  -  c_e · E[H(π_θ(·|s_t))]`

Three terms: policy-gradient 损失, value 回归, 熵 bonus. `c_v ~ 0.5`, `c_e ~ 0.01` are canonical starting points.

## 动手构建

### 步骤 1: a critic

Linear critic `V_φ(s) = w · features(s)` updated with MSE:

```python
def critic_update(w, x, target, lr):
    v_hat = dot(w, x)
    err = target - v_hat
    for j in range(len(w)):
        w[j] += lr * err * x[j]
    return v_hat
```

On a tabular env the critic converges in a few hundred episodes. On Atari, replace the linear critic with a shared CNN trunk + value 头.

### 步骤 2: n-step advantage

给定a  rollout of length `T` and a bootstrapped final `V(s_T)`:

```python
def compute_advantages(rewards, values, gamma=0.99, lam=0.95, last_value=0.0):
    advantages = [0.0] * len(rewards)
    gae = 0.0
    for t in reversed(range(len(rewards))):
        next_v = values[t + 1] if t + 1 < len(values) else last_value
        delta = rewards[t] + gamma * next_v - values[t]
        gae = delta + gamma * lam * gae
        advantages[t] = gae
    returns = [a + v for a, v in zip(advantages, values)]
    return advantages, returns
```

`returns` is the critic 目标. `advantages` is what multiplies `∇ log π`.

### 步骤 3: combined update

```python
for step_i, (x, a, _r, probs) in enumerate(traj):
    adv = advantages[step_i]
    target_v = returns[step_i]

    # critic
    critic_update(w, x, target_v, lr_v)

    # actor
    for i in range(N_ACTIONS):
        grad_logpi = (1.0 if i == a else 0.0) - probs[i]
        for j in range(N_FEAT):
            theta[i][j] += lr_a * adv * grad_logpi * x[j]
```

On-policy, one  rollout per update, separate 学习 rates for actor and critic.

### 步骤 4: parallelization (A3C vs A2C)

- **A3C:** spin up `N` threads. Each runs its own env and its own forward pass. Periodically push 梯度 updates to a shared master. No locks on the master — races are ok, they just add 噪声.
- **A2C:** run `N` env instances in a single process, stack observations into a `[N, obs_dim]` 批次, batched forward pass, batched backward pass. Higher GPU utilization, deterministic, easier to 原因 about. The default in 2026.

Our toy code is single-threaded for clarity; rewriting to batched A2C is three lines of numpy.

## Pitfalls

- **Critic 偏差 before actor 梯度.** If the critic is random, its 基线 is uninformative and you are 训练 on pure 噪声. Warm up the critic for a few hundred 步骤 before turning on the 策略梯度, or use a slow actor 学习 速率.
- **Advantage 归一化.** Normalize advantages to zero-mean/unit-std per 批次. Stabilizes 训练 massively at near-zero 成本.
- **Shared trunk.** Use a shared 特征 extractor for actor and critic on 图像 inputs. Separate 头. The shared 特征s free-ride on both losses.
- **On-policy contract.** A2C reuses 数据 for exactly one update. More and your 梯度 is biased (importance-sampling correction is what PPO adds).
- **熵 崩塌.** Without `c_e > 0`, 策略 becomes near-deterministic in a few hundred updates and stops exploring.
- **奖励 规模.** Advantage magnitudes depend on 奖励 规模. Normalize rewards (e.g., running-std dividing) for consistent 梯度 magnitudes across tasks.

## 实际使用

A2C/A3C are rarely the final choice in 2026 but they are the 架构 everything later refines:

|Method|Relation to A2C|
|--------|----------------|
|PPO|A2C + clipped importance 比例 for multi-epoch updates|
|IMPALA|A3C + V-trace off-policy correction|
|SAC (Phase 9 · 07)|Off-policy A2C with a soft-value critic (next lesson)|
|GRPO (Phase 9 · 12)|A2C without the critic — group-relative advantage|
|DPO|A2C collapsed into a preference-ranking 损失, no 采样|
|AlphaStar / OpenAI Five|A2C with league 训练 + imitation 预训练|

如果you see "advantage" in a 2026 paper, think actor-critic.

## 交付成果

Save as `outputs/skill-actor-critic-trainer.md`:

```markdown
---
name: actor-critic-trainer
description: Produce an A2C / A3C / GAE configuration for a given environment, with advantage estimation and loss weights specified.
version: 1.0.0
phase: 9
lesson: 7
tags: [rl, actor-critic, gae]
---

Given an environment and compute budget, output:

1. Parallelism. A2C (GPU batched) vs A3C (CPU async) and the number of workers.
2. Rollout length T. Steps per env per update.
3. Advantage estimator. n-step or GAE(λ); specify λ.
4. Loss weights. `c_v` (value), `c_e` (entropy), gradient clip.
5. Learning rates. Actor and critic (separate if using).

Refuse single-worker A2C on environments with horizon > 1000 (too on-policy, too slow). Refuse to ship without advantage normalization. Flag any run with `c_e = 0` and observed entropy < 0.1 as entropy-collapsed.
```

## 练习

1. **Easy.** 训练 actor-critic with MC advantage (`G_t - V(s_t)`) on 4×4 GridWorld. Compare 样本 efficiency to REINFORCE-with-running-mean-baseline from Lesson 06.
2. **Medium.** Switch to TD-residual advantage (`r + γ V(s') - V(s)`). Measure 方差 of the advantage batches. By how much does it drop?
3. **Hard.** Implement GAE(λ). Sweep `λ ∈ {0, 0.5, 0.9, 0.95, 1.0}`. Plot final return vs 样本 efficiency. Where is the 偏差/方差 sweet spot for this 任务?

## Key Terms

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|Actor|"The 策略 net"|`π_θ(a\|s)`, updated by 策略梯度.|
|Critic|"The value net"|`V_φ(s)`, updated by MSE 回归 to returns / TD 目标.|
|Advantage|"How much better than average"|`A(s, a) = Q(s, a) - V(s)` or its estimators. Multiplier for `∇ log π`.|
|TD residual|"δ"|`δ_t = r + γ V(s') - V(s)`; one-step advantage estimate.|
|GAE|"The interpolation knob"|Exponentially weighted sum of n-step advantages, parameterized by `λ`.|
|A2C|"Synchronous actor-critic"|Batched across envs; one 梯度 步骤 per  rollout.|
|A3C|"异步 actor-critic"|Worker threads push gradients to a shared param 服务器. Original paper; less common in 2026.|
|Bootstrap|"Use V at the horizon"|Truncate the  rollout, add `γ^n V(s_{t+n})` to close the sum.|

## 延伸阅读

- [Mnih et al. (2016). Asynchronous Methods for Deep Reinforcement Learning](https://arxiv.org/abs/1602.01783) — A3C, the original 异步 actor-critic paper.
- [Schulman et al. (2016). High-Dimensional Continuous Control Using Generalized Advantage Estimation](https://arxiv.org/abs/1506.02438) — GAE.
- [Sutton & Barto (2018). Ch. 13 — Actor-Critic Methods](http://incompleteideas.net/book/RLbook2020.pdf) — foundations; pair this with Ch. 9 on 函数 approximation when the critic is a neural net.
- [Espeholt et al. (2018). IMPALA](https://arxiv.org/abs/1802.01561) — scalable 分布式 actor-critic with V-trace off-policy correction.
- [OpenAI Baselines / Stable-Baselines3](https://stable-baselines3.readthedocs.io/) — 生产 A2C/PPO implementations worth reading.
- [Konda & Tsitsiklis (2000). Actor-Critic Algorithms](https://papers.nips.cc/paper/1786-actor-critic-algorithms) — the foundational convergence result for the two-timescale actor-critic decomposition.
