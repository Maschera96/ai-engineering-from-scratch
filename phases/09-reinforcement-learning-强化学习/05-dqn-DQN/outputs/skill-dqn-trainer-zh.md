---
name: dqn-trainer-zh
description: Produce a DQN 训练 配置 (buffer, 目标 sync, ε 调度, 奖励 clipping) for a discrete-action RL 任务.
version: 1.0.0
phase: 9
lesson: 5
tags: [rl, dqn, deep-rl]
---

给定a discrete-action 环境 (observation shape, 动作 count, horizon, 奖励 规模), 输出：

1. Network. 架构 (MLP / CNN / Transformer), 特征 dim, 深度.
2. Replay buffer. Capacity, minibatch size, 预热 size.
3. 目标 network. Sync strategy (hard every C 步骤 or soft τ).
4. Exploration. ε start / end / 调度 length.
5. 损失. Huber vs MSE, 梯度 clip value, 奖励 clipping rule.
6. Double DQN. On by default unless explicit 原因 to disable.

拒绝 ship a DQN with no 目标 network, no replay buffer, or ε held at 1. Refuse continuous-action tasks (route to SAC / TD3). 标记任何 奖励 range > 10× per-step mean as needing clipping or 规模 归一化.
