---
name: td-agent-zh
description: 选择between Q-learning, SARSA, Expected SARSA for a tabular or small-特征 RL 任务.
version: 1.0.0
phase: 9
lesson: 4
tags: [rl, td-learning, q-learning, sarsa]
---

给定a tabular or small-特征 环境, 输出：

1. Algorithm. Q-learning / SARSA / Expected SARSA / n-step variant. 用一句话说明原因 tied to on-policy vs off-policy and 方差.
2. Hyperparameters. α, γ, ε, 衰减 调度.
3. Initialization. Q_0 value (optimistic vs zero) and justification.
4. Convergence diagnostic. 目标 学习 曲线, `|Q - Q*|` check if DP is possible.
5. Deployment caveat. How will exploration behave at 推理? Is SARSA's conservatism needed?

拒绝 apply tabular TD to 状态 spaces > 10⁶. 拒绝 ship a Q-learning 智能体 without a max-bias caveat. 标记任何 智能体 训练后的 with ε held at 1.0 throughout (no exploitation phase).
