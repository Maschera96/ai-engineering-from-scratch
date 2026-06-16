---
name: actor-critic-trainer-zh
description: Produce an A2C / A3C / GAE configuration for a given 环境, with advantage estimation and 损失 权重 specified.
version: 1.0.0
phase: 9
lesson: 7
tags: [rl, actor-critic, gae]
---

给定an 环境 and 计算 预算, 输出：

1. Parallelism. A2C (GPU batched) vs A3C (CPU 异步) and the number of workers.
2.  rollout length T. 步骤 per env per update.
3. Advantage estimator. n-step or GAE(λ); specify λ.
4. 损失 权重. `c_v` (value), `c_e` (熵), 梯度 clip.
5. 学习 rates. Actor and critic (separate if using).

拒绝single-worker A2C on environments with horizon > 1000 (too on-policy, too slow). 拒绝 ship without advantage 归一化. 标记任何 run with `c_e = 0` and observed 熵 < 0.1 as entropy-collapsed.
