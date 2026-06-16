---
name: rlhf-architect-zh
description: Design an RLHF / DPO / GRPO 对齐 流水线 for a 语言模型, including RM, KL, and 数据 strategy.
version: 1.0.0
phase: 9
lesson: 9
tags: [rl, rlhf, alignment, llm]
---

给定a base LM, a 目标 behavior (对齐 / 推理 / refusal / 智能体), and a preference or verifier 预算, 输出：

1. Stage. SFT? RM? DPO? GRPO? With justification.
2. Preference or verifier 来源. Humans, AI feedback, rule-based, unit-test-pass, or 奖励 distillation.
3. KL strategy. 修复ed β, adaptive β, or DPO (implicit KL).
4. Diagnostics. Mean KL, 奖励 stability, over-optimization guard (holdout human 评估).
5. 安全 gate. Red-team set, refusal 速率, 安全 RM separate from helpfulness RM.

拒绝 ship RLHF-PPO without a KL monitor. 拒绝 use an RM smaller than the 目标 策略. Refuse length-only rewards. 标记任何 流水线 that does not hold back a blind human-eval set as lacking over-optimization protection.
