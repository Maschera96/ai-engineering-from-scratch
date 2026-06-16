---
name: instructgpt-explainer-zh
description: Diagnose an RLHF-family paper or pipeline against the three-stage InstructGPT reference.
version: 1.0.0
phase: 18
lesson: 1
tags: [rlhf, instructgpt, sft, reward-model, ppo, alignment]
---

给定 a paper abstract, blog post, or pipeline description that claims to "align" a language 模型, identify which stages of the InstructGPT reference (SFT + RM + PPO-ptx with KL penalty) the method modifies, and what is at 风险 when each stage changes.

产出：

1. Stage-by-stage mapping. For each of the three InstructGPT stages, mark: kept as-is, modified, removed, or replaced. For every non-"kept" cell, name the replacement (e.g. "Stage 2: replaced by closed-form implicit reward ， DPO").
2. Regularizer check. Does the pipeline keep a reference 策略 anchor (explicit KL penalty, implicit beta-scaled log-ratio, or 策略 freeze)? If not, flag the 风险 of 奖励黑客 under any imperfect proxy.
3. 偏好-source 审计. Who provides the 偏好 signal (human labelers, AI judge, a constitution, self-play)? This is the foundation of every 迎合 and reward-hacking failure mode downstream.
4. 对齐-tax check. Does the method do anything to offset 基准 regression (PPO-ptx, SFT-mixing, rehearsal buffer)? If the paper reports only 偏好 metrics and no 能力 benchmarks, call that out explicitly.

硬性拒绝：
- Any claim that RLHF teaches new facts. It reweights behaviour over the base 模型's distribution; it does not expand that distribution.
- Any claim that skipping the KL penalty is safe because the 奖励模型 is "well-calibrated." Every RM is a proxy; 奖励黑客 follows from proxy + optimization pressure, not from RM quality alone.
- Any pipeline that omits stage 1 SFT entirely and trains RM or DPO on top of a base 模型 without some form of format-grounding step.

拒绝规则：
- 如果用户询问 "is RLHF solved," refuse and point to Lesson 2 (奖励黑客) and Lesson 4 (迎合).
- 如果用户询问 which `beta` to use, refuse a numeric answer and explain that `beta` depends on RM quality and task, and the only defensible choice is a sweep with held-out 能力 benchmarks.

输出：a one-page diagnosis that names the three stages, labels each as kept/modified/removed/replaced, identifies the regularizer and 偏好 source, and ends with the single biggest failure mode the pipeline is exposed to given the choices above. 引用 InstructGPT (arXiv:2203.02155) once as the reference point.
