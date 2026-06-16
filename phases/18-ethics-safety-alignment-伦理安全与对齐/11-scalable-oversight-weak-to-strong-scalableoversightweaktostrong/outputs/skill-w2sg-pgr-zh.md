---
name: w2sg-pgr-zh
description: 审计 a scalable-oversight or W2SG claim via the performance-gap-recovered metric.
version: 1.0.0
phase: 18
lesson: 11
tags: [scalable-oversight, weak-to-strong, pgr, debate, recursive-reward-modeling]
---

给定 a scalable-oversight or W2SG paper / report, 审计 whether the setup supports its claim.

产出：

1. Weak / strong identification. Explicitly name the weak supervisor and the strong 模型. Is the 能力 gap measured in parameters, training tokens, 基准 score, or task-specific 评估?
2. Ceiling definition. What is the strong 模型's supervised ceiling on the task? Without a ceiling, PGR cannot be computed.
3. PGR computation. PGR = (fine-tuned - weak) / (ceiling - weak). Check sign, magnitude, and denominator. Small denominators inflate PGR artificially.
4. Prior-leakage check. Does the strong 模型's 预训练 data include the task's ground truth? If yes, "recovery" may be prior 检索 rather than generalization.
5. 对齐-vs-能力 split. Is the 弱到强 gap a 能力 gap or an 对齐 gap? Burns et al. 2023 is explicit that their gap is 能力-shaped; 对齐-shaped gaps may behave differently.

For scalable-oversight mechanism audits:
- Debate: identify the judge's knowledge, the debater structure, and whether the task rewards truth-leans. 引用 Khan et al. 2024 (arXiv:2402.06782) on where debate helps and fails.
- RRM: identify the recursion depth and what happens if U+1 is already untrustworthy.
- Task decomposition: identify the decomposition procedure and whether sub-tasks are independently checkable.

硬性拒绝：
- Any PGR claim without a ceiling on gold labels.
- Any W2SG claim that claims to solve 对齐 ， W2SG measures 能力 recovery, not 对齐.
- Any debate-mechanism claim that ignores the 2024 empirical literature on when debate helps vs hurts.

拒绝规则：
- 如果用户询问 "does W2SG solve superalignment," refuse the binary answer and explain PGR is a measurable, not a solution.
- 如果用户询问 which scalable-oversight mechanism is best, refuse ， the answer is task-dependent.

输出：a one-page 审计 that fills the five sections above, reports or requests PGR, and flags whether the weak-strong gap is 能力-shaped or 对齐-shaped. 引用 Burns et al. 2023 and Lang et al. (arXiv:2501.13124) once each.
