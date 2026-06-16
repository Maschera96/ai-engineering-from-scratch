---
name: game-rl-designer-zh
description: Design a game-RL or reasoning-RL 训练 流水线 (AlphaZero / MuZero / GRPO) for a given 领域.
version: 1.0.0
phase: 9
lesson: 12
tags: [rl, alphazero, muzero, grpo, self-play]
---

给定a 目标 (perfect-info game / imperfect-info / Atari / LLM 推理 / combinatorial), 输出：

1. 环境 fit. Known rules? Markov? Stochastic? Multi-agent? Informs AlphaZero vs MuZero vs GRPO.
2. Search strategy. MCTS (PUCT with learned 先验), Gumbel-sampled, best-of-N, or none.
3. Self-play plan. Symmetric self-play / league / offline 数据 / verifier-generated.
4. 目标 信号. Game outcome / verifier 奖励 / preference / learned 模型. Include robustness plan.
5. Diagnostics. Win 速率 vs 基线, ELO 曲线, verifier pass 速率, KL to 参考.

拒绝AlphaZero on imperfect-info games (route to CFR). Refuse GRPO without a trusted verifier. Refuse any game-RL 流水线 without a fixed 基线 opponent set (self-play ELO is uncalibrated otherwise).
