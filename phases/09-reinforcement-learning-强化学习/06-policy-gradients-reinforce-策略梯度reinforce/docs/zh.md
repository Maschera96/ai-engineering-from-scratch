# 策略 梯度 — REINFORCE from Scratch

> Stop estimating value. Parameterize the 策略 directly, 计算 the 梯度 of expected return, 步骤 uphill. Williams (1992) wrote it in one theorem. It is why PPO, GRPO, and every LLM RL 循环 exist.

**类型：** Build
**语言：** Python
**先修：** Phase 3 · 03 (Backpropagation), Phase 9 · 03 (Monte Carlo), Phase 9 · 04 (TD 学习)
**时间：** 约 75 分钟

## 问题

Q-learning and DQN parameterize the *value* 函数. You pick actions by `argmax Q`. That is fine for discrete actions and discrete states. It breaks when actions are continuous (which `argmax` over a 10-dimensional torque?) or when you want a stochastic 策略 (`argmax` is deterministic by construction).

策略梯度s parameterize the *策略* instead. `π_θ(a | s)` is a neural net that outputs a 分布 over actions. 样本 from it to act. 计算 the 梯度 of expected return with respect to `θ`. 步骤 uphill. No `argmax`. No Bellman recursion. Just 梯度 ascent on `J(θ) = E_{π_θ}[G]`.

这个REINFORCE theorem (Williams 1992) tells you this 梯度 is computable: `∇J(θ) = E_π[ G · ∇_θ log π_θ(a | s) ]`. Run an episode. 计算 the return. Multiply by `∇ log π_θ(a | s)` at every 步骤. Average. Gradient-ascent. Done.

每个LLM-RL algorithm in 2026 — PPO, DPO, GRPO — is a refinement of REINFORCE. Understanding it in your fingers is the prerequisite for the rest of this phase, and for Phase 10 · 07 (RLHF implementation) and Phase 10 · 08 (DPO).

## 概念

![Policy gradient: softmax policy, log-π gradient, return-weighted update](../assets/policy-gradient.svg)

**The 策略梯度 theorem.** For any 策略 `π_θ` parameterized by `θ`:

`∇J(θ) = E_{τ ~ π_θ}[ Σ_{t=0}^{T} G_t · ∇_θ log π_θ(a_t | s_t) ]`

where `G_t = Σ_{k=t}^{T} γ^{k-t} r_{k+1}` is the discounted return from 步骤 `t`. The expectation is over full trajectories `τ` sampled from `π_θ`.

**The proof is short.** Differentiate `J(θ) = Σ_τ P(τ; θ) G(τ)` under the expectation. Use `∇P(τ; θ) = P(τ; θ) ∇ log P(τ; θ)` (the log-derivative trick). Factor `log P(τ; θ) = Σ log π_θ(a_t | s_t) + environment terms that do not depend on θ`. The 环境 terms vanish. Two lines of algebra give you the theorem.

**方差 reduction tricks.** Vanilla REINFORCE has murderous 方差 — returns are noisy, `∇ log π` is noisy, their product is very noisy. Two standard fixes:

1. **基线 subtraction.** Replace `G_t` with `G_t - b(s_t)` for any 基线 `b(s_t)` that does not depend on `a_t`. Unbiased because `E[b(s_t) · ∇ log π(a_t | s_t)] = 0`. Typical choice: `b(s_t) = V̂(s_t)` learned by a critic → actor-critic (Lesson 07).
2. **Reward-to-go.** Replace `Σ_t G_t · ∇ log π_θ(a_t | s_t)` with `Σ_t G_t^{from t} · ∇ log π_θ(a_t | s_t)`. Only future returns matter for a given 动作 — past rewards contribute zero-mean 噪声.

Combined, you get:

`∇J ≈ (1/N) Σ_{i=1}^{N} Σ_{t=0}^{T_i} [ G_t^{(i)} - V̂(s_t^{(i)}) ] · ∇_θ log π_θ(a_t^{(i)} | s_t^{(i)})`

which is REINFORCE with a 基线 — the direct ancestor of A2C (Lesson 07) and PPO (Lesson 08).

**Softmax 策略 parameterization.** For discrete actions, the standard choice:

`π_θ(a | s) = exp(f_θ(s, a)) / Σ_{a'} exp(f_θ(s, a'))`

where `f_θ` is any neural net that outputs a 分数 per 动作. The 梯度 has a clean form:

`∇_θ log π_θ(a | s) = ∇_θ f_θ(s, a) - Σ_{a'} π_θ(a' | s) ∇_θ f_θ(s, a')`

i.e., 分数 of the taken 动作 minus its expected value under the 策略.

**Gaussian 策略 for continuous actions.** `π_θ(a | s) = N(μ_θ(s), σ_θ(s))`. `∇ log N(a; μ, σ)` has a closed form. That is all Phase 9 · 07's SAC needs.

```figure
policy-gradient-landscape
```

## 动手构建

### 步骤 1: softmax 策略 network

```python
def policy_logits(theta, state_features):
    return [dot(theta[a], state_features) for a in range(N_ACTIONS)]

def softmax(logits):
    m = max(logits)
    exps = [exp(l - m) for l in logits]
    Z = sum(exps)
    return [e / Z for e in exps]
```

使用a linear 策略 (one 权重 vector per 动作) for a tabular env. For Atari, swap in a CNN and keep the softmax 头.

### 步骤 2: 采样 and log-probability

```python
def sample_action(probs, rng):
    x = rng.random()
    cum = 0
    for a, p in enumerate(probs):
        cum += p
        if x <= cum:
            return a
    return len(probs) - 1

def log_prob(probs, a):
    return log(probs[a] + 1e-12)
```

### 步骤 3:  rollout with log-probs captured

```python
def rollout(theta, env, rng, gamma):
    trajectory = []
    s = env.reset()
    while not done:
        logits = policy_logits(theta, s)
        probs = softmax(logits)
        a = sample_action(probs, rng)
        s_next, r, done = env.step(s, a)
        trajectory.append((s, a, r, probs))
        s = s_next
    return trajectory
```

### 步骤 4: REINFORCE update

```python
def reinforce_step(theta, trajectory, gamma, lr, baseline=0.0):
    returns = compute_returns(trajectory, gamma)
    for (s, a, _, probs), G in zip(trajectory, returns):
        advantage = G - baseline
        grad_log_pi_a = [-p for p in probs]
        grad_log_pi_a[a] += 1.0
        for i in range(N_ACTIONS):
            for j in range(len(s)):
                theta[i][j] += lr * advantage * grad_log_pi_a[i] * s[j]
```

这个梯度 `∇ log π(a|s) = e_a - π(·|s)` (onehot of `a` minus 概率) is the heart of softmax 策略梯度s. Burn it into muscle 内存.

### 步骤 5: baselines

一个running mean of `G` over recent episodes is enough 方差 reduction to get a 4×4 GridWorld running; it takes ~500 episodes to converge. Upgrade the 基线 to a learned `V̂(s)` and you get actor-critic.

## Pitfalls

- **Exploding gradients.** Returns can be huge. Always normalize `G` to `~N(0, 1)` across the 批次 before multiplying by `∇ log π`.
- **熵 崩塌.** The 策略 converges to a near-deterministic 动作 too early, stops exploring, gets stuck. 修复: add 熵 bonus `β · H(π(·|s))` to the 目标.
- **High 方差.** Vanilla REINFORCE needs thousands of episodes. A critic 基线 (Lesson 07) or TRPO/PPO's trust region (Lesson 08) is the standard fix.
- **样本 inefficiency.** On-policy means you throw away every transition after one update. Off-policy corrections via importance 采样 bring back 数据, at the 成本 of 方差 (PPO's 比例 is a clipped IS 权重).
- **Non-stationary gradients.** The same 梯度 from 100 episodes ago uses old `π`. On-policy methods update every few rollouts for this 原因.
- **Credit assignment.** Without reward-to-go, past rewards contribute 噪声. Always use reward-to-go.

## 实际使用

In 2026, REINFORCE is rarely run directly but its 梯度 formula is everywhere:

|Use case|Derived method|
|----------|---------------|
|Continuous control|PPO / SAC with Gaussian 策略|
|LLM RLHF|PPO with KL penalty, running on token-level 策略|
|LLM 推理 (DeepSeek)|GRPO — REINFORCE with group-relative 基线, no critic|
|Multi-agent|Centralized-critic REINFORCE (MADDPG, COMA)|
|Discrete 动作 robotics|A2C, A3C, PPO|
|Preference-only settings|DPO — REINFORCE rewritten as a preference-likelihood 损失, no 采样|

当you read `loss = -advantage * log_prob` in a 2026 训练 script, that is REINFORCE with a 基线. Entire papers (DPO, GRPO, RLOO) are variance-reduction tricks on top of this one line.

## 交付成果

Save as `outputs/skill-policy-gradient-trainer.md`:

```markdown
---
name: policy-gradient-trainer
description: Produce a REINFORCE / actor-critic / PPO training config for a given task and diagnose variance issues.
version: 1.0.0
phase: 9
lesson: 6
tags: [rl, policy-gradient, reinforce]
---

Given an environment (discrete / continuous actions, horizon, reward stats), output:

1. Policy head. Softmax (discrete) or Gaussian (continuous) with parameter counts.
2. Baseline. None (vanilla), running mean, learned `V̂(s)`, or A2C critic.
3. Variance controls. Reward-to-go on by default, return normalization, gradient clip value.
4. Entropy bonus. Coefficient β and decay schedule.
5. Batch size. Episodes per update; on-policy data freshness contract.

Refuse REINFORCE-no-baseline on horizons > 500 steps. Refuse continuous-action control with a softmax head. Flag any run with `β = 0` and observed policy entropy < 0.1 as entropy-collapsed.
```

## 练习

1. **Easy.** Implement REINFORCE on 4×4 GridWorld with a linear softmax 策略. 训练 for 1,000 episodes without a 基线. Plot the 学习 曲线; measure 方差 (std of returns).
2. **Medium.** Add a running-mean 基线. 训练 again. Compare 样本 efficiency and 方差 to the vanilla run. By how much does the 基线 reduce 步骤 to convergence?
3. **Hard.** Add an 熵 bonus `β · H(π)`. Sweep `β ∈ {0, 0.01, 0.1, 1.0}`. Plot final return and 策略 熵. Where is the sweet spot on this 任务?

## Key Terms

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|策略梯度|"训练 the 策略 directly"|`∇J(θ) = E[G · ∇ log π_θ(a\|s)]`; derived from the log-derivative trick.|
|REINFORCE|"The original PG algorithm"|Williams (1992); Monte Carlo returns multiplied by log-策略梯度.|
|Log-derivative trick|"分数 函数 estimator"|`∇P(τ;θ) = P(τ;θ) · ∇ log P(τ;θ)`; makes gradients of expectations tractable.|
|基线|"方差 reduction"|Any `b(s)` subtracted from `G`; unbiased because `E[b · ∇ log π] = 0`.|
|Reward-to-go|"Only future returns count"|`G_t^{from t}` instead of the full `G_0`; correct and lower-variance.|
|熵 bonus|"Encourage exploration"|`+β · H(π(·\|s))` term keeps the 策略 from collapsing.|
|On-policy|"训练 on what you just saw"|梯度 expectation is w.r.t. the current 策略 — cannot reuse old 数据 directly.|
|Advantage|"How much better than average"|`A(s, a) = G(s, a) - V(s)`; the signed quantity REINFORCE-with-baseline multiplies.|

## 延伸阅读

- [Williams (1992). Simple Statistical Gradient-Following Algorithms for Connectionist Reinforcement Learning](https://link.springer.com/article/10.1007/BF00992696) — the original REINFORCE paper.
- [Sutton et al. (2000). Policy Gradient Methods for Reinforcement Learning with Function Approximation](https://papers.nips.cc/paper_files/paper/1999/hash/464d828b85b0bed98e80ade0a5c43b0f-Abstract.html) — the modern policy-gradient theorem with 函数 approximation.
- [Sutton & Barto (2018). Ch. 13 — Policy Gradient Methods](http://incompleteideas.net/book/RLbook2020.pdf) — textbook presentation.
- [OpenAI Spinning Up — VPG / REINFORCE](https://spinningup.openai.com/en/latest/algorithms/vpg.html) — clear pedagogical exposition with PyTorch code.
- [Peters & Schaal (2008). Reinforcement Learning of Motor Skills with Policy Gradients](https://homes.cs.washington.edu/~todorov/courses/amath579/reading/PolicyGradient.pdf) — variance-reduction and the natural-gradient view that connects REINFORCE to the trust-region family (TRPO, PPO).
