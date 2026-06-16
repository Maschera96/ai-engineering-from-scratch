---
name: skill-sampling-strategy-zh
description: Choose the right 采样 method for generation, estimation, 或 inference
version: 1.0.0
phase: 1
lesson: 16
tags: [sampling, mcmc, generation]
---

# 采样 Strategy Selection

How to pick the right 采样 method for text generation, 贝叶斯 inference, 蒙特卡洛 estimation, 和 training.

## Decision Checklist

1. Are you generating 输出 (text, images) 或 estimating a quantity (integral, 期望)?
2. Can you 样本 directly from the target 分布, 或 only evaluate its density?
3. Is the target 分布 discrete 或 continuous?
4. What 维度 is the 样本 空间? Low (< 5), medium (5-100), 或 high (> 100)?
5. Do you need exact 样本 或 approximate ones?
6. Do you need 梯度 through the 采样 operation?

## When to use each method

| Method | When to use | Complexity | Exact? |
|---|---|---|---|
| Direct 采样 | You have the CDF 或 can use a library 函数 | O(1) per 样本 | Yes |
| Inverse CDF | Known closed-form CDF inverse (exponential, Cauchy) | O(1) per 样本 | Yes |
| Box-Muller | Need normal 样本 without a library | O(1) per 样本 | Yes |
| Rejection 采样 | Can evaluate target PDF, low 维度 (1-3) | O(1/acceptance) per 样本 | Yes |
| Importance 采样 | Need expectations, not individual 样本 | O(n) for n 样本 | Approximate |
| Stratified 采样 | 蒙特卡洛 estimation, want lower 方差 | O(n) for n 样本 | Approximate |
| Metropolis-Hastings | High-dimensional, can evaluate unnormalized density | O(1) per step + burn-in | Asymptotically |
| Gibbs 采样 | Can 样本 from each conditional 分布 | O(d) per full sweep | Asymptotically |
| HMC/NUTS | High-dimensional continuous, smooth density | O(L * d) per step | Asymptotically |
| Temperature 采样 | LLM text generation, control creativity | O(V) for vocab size V | N/A |
| Top-k 采样 | LLM generation, remove unlikely tokens | O(V log k) | N/A |
| Top-p (nucleus) | LLM generation, adaptive c和idate set | O(V log V) | N/A |
| Reparameterization | Need 梯度 through Gaussian 采样 (VAEs) | O(d) | Yes |
| Gumbel-Softmax | Need 梯度 through categorical 采样 | O(k) for k classes | Approximate |

## LLM generation settings

| Use case | Temperature | Top-p | Top-k | Notes |
|---|---|---|---|---|
| Factual Q&A | 0.0 (greedy) | -- | -- | Deterministic, no r和omness |
| Code generation | 0.2-0.5 | 0.9 | -- | Low creativity, high coherence |
| General chat | 0.7 | 0.9 | -- | Balanced |
| Creative writing | 0.9-1.2 | 0.95 | -- | Higher diversity |
| Brainstorming | 1.0-1.5 | 0.95 | -- | Maximum diversity, may lose coherence |

Temperature 和 top-p can be combined. Apply temperature first (scale logits), then apply top-p filtering.

## MCMC method selection

| Property | Metropolis-Hastings | Gibbs | HMC/NUTS |
|---|---|---|---|
| Dimension | Any | Any (best < 100) | High (100+) |
| Requires conditionals | No | Yes | No |
| Requires 梯度 | No | No | Yes |
| Acceptance rate | Tune to ~23% | Always 100% | Tune to ~65% |
| Correlation | High (随机 walk) | Moderate | Low |
| Burn-in | Long | Moderate | Short |
| Best for | Exploration, simple 模型 | Conjugate 模型, 贝叶斯 networks | Continuous posteriors, deep probabilistic 模型 |

## Common mistakes

- Using rejection 采样 in high 维度. Acceptance rate drops exponentially 与 维度. Above 5 维度, switch to MCMC.
- Setting MCMC proposal 方差 too high 或 too low. Too high: most proposals rejected, chain stuck. Too low: all proposals accepted, chain moves slowly. Target ~23% acceptance for 随机 walk MH.
- Forgetting burn-in. The first N 样本 from MCMC are biased by the starting point. Discard at least 1000 steps (or more for 复数 分布).
- Using importance 采样 与 a proposal very different from the target. A few 样本 get enormous weights, making the estimate unreliable. Monitor the effective 样本 size: ESS = (sum w_i)^2 / sum(w_i^2).
- Using temperature > 0 for tasks that need deterministic 输出 (e.g., 分类, structured extraction). Use greedy (T=0) 或 beam search instead.
- Not combining top-p 与 temperature. Temperature alone does not remove garbage tokens from the long tail. Top-p does.
- Backpropagating through a st和ard 采样 operation. Use reparameterization trick for continuous (Gaussian) 和 Gumbel-Softmax for discrete (categorical).

## Quick reference: 方差 reduction techniques

| Technique | How it works | 方差 reduction |
|---|---|---|
| Stratified 采样 | Divide 空间 into strata, 样本 each | Always <= st和ard MC |
| Antithetic variates | Use both U 和 1-U | Works for monotone 函数 |
| Control variates | Subtract a known-均值 variable | Proportional to 相关性 |
| Importance 采样 | Reweight 样本 from a better proposal | Depends on proposal quality |
| Latin hypercube | Stratify each 维度 independently | Better than stratified in high-d |
