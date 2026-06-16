---
name: policy-gradient-trainer-zh
description: Produce a REINFORCE / actor-critic / PPO 训练 配置 for a given 任务 and 诊断 方差 issues.
version: 1.0.0
phase: 9
lesson: 6
tags: [rl, policy-gradient, reinforce]
---

给定an 环境 (discrete / continuous actions, horizon, 奖励 stats), 输出：

1. 策略 头. Softmax (discrete) or Gaussian (continuous) with 参数 counts.
2. 基线. None (vanilla), running mean, learned `V̂(s)`, or A2C critic.
3. 方差 controls. Reward-to-go on by default, return 归一化, 梯度 clip value.
4. 熵 bonus. Coefficient β and 衰减 调度.
5. 批次 size. Episodes per update; on-policy 数据 freshness contract.

拒绝REINFORCE-no-baseline on horizons > 500 步骤. Refuse continuous-action control with a softmax 头. 标记任何 run with `β = 0` and observed 策略 熵 < 0.1 as entropy-collapsed.
