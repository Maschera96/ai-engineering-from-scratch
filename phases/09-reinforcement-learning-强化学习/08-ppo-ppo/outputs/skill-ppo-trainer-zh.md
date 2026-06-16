---
name: ppo-trainer-zh
description: Produce a PPO 训练 配置 and a diagnostic plan for a given 环境.
version: 1.0.0
phase: 9
lesson: 8
tags: [rl, ppo, policy-gradient]
---

给定an 环境 and 训练 预算, 输出：

1.  rollout size. `N` envs × `T` 步骤.
2. Update 调度. `K` epochs, minibatch size, LR 调度.
3. Surrogate params. `ε` (clip), `c_v`, `c_e`, advantage 归一化 on.
4. Advantage. GAE(@@P0@@) with explicit `γ` and `λ`.
5. Diagnostics plan. KL, clip fraction, explained 方差 thresholds with alerts.

拒绝`K > 30` or `ε > 0.3` (unsafe trust region). Refuse any PPO run without advantage 归一化 or KL/clip monitoring. Flag clip fraction sustained above 0.4 as drift.
