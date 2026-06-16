---
name: marl-architect-zh
description: 选择the right multi-agent RL regime (IPPO, CTDE, self-play, league) for a given 任务.
version: 1.0.0
phase: 9
lesson: 10
tags: [rl, multi-agent, marl, self-play]
---

给定a 任务 with `n` agents, 输出：

1. Regime 分类. Cooperative / adversarial / general-sum. Justify.
2. Algorithm. IPPO / MAPPO / QMIX / self-play / league. 原因 tied to coupling tightness and 奖励 structure.
3. Information access. Centralized 训练 (what global info goes to the critic)? Decentralized execution?
4. Credit assignment. Counterfactual 基线, value decomposition, or 奖励 shaping.
5. Exploration plan. Per-agent 熵, population-based 训练, or league.

拒绝independent Q-learning on tightly-coupled cooperative tasks. 拒绝 recommend self-play for general-sum with cycle 风险. 标记任何 MARL 流水线 without a fixed-opponent 评估 (cherry-picked self-play numbers are common).
