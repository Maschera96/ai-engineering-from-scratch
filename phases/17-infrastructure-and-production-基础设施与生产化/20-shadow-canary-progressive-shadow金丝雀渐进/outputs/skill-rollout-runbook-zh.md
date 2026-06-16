---
name: rollout-runbook-zh
description: 设计a shadow → 金丝雀 → A/B → 100% rollout 计划 for a new LLM 模型 或 提示词 template, with five 金丝雀 gates, noise-floor-aware thresholds, and a seconds-fast 回滚 path.
version: 1.0.0
phase: 17
lesson: 20
tags: [rollout, canary, shadow, progressive-delivery, feature-flags, argo-rollouts, flagger, kserve]
---

给定a candidate change (new 模型, new 提示词 template, new 路由器 策略), 基线生产化 指标, and 风险 tolerance, produce a rollout 运行手册.

产出：

1. Shadow 计划. Duration (24-72 hours). 指标 logged: 输出, 词元 counts, 延迟, refusal, error. Alert on: >20% 成本 shift, >30% 输出 length shift, any schema violation.
2. 金丝雀 progression. Stages (1% → 10% → 25% → 50% → 75% → 100%). Duration per stage (30m-24h based on 流量 volume; ensure each stage has enough data for statistical confidence).
3. Five gates. Specify the exact thresholds for 延迟 P99, 成本/请求, error/refusal, 输出-length P99, thumbs-down rate. Set above noise floor (expect 15% irreducible 方差).
4. Tooling. Name the rollout controller (Argo Rollouts, Flagger, KServe) and 这个功能 flag system for instant 回滚.
5. 回滚 path. Document the three actions: flip flag → revert pinned digest → verify. Target time: under 60 seconds end to end.
6. Skip A/B? 说明理由. Improved-variant changes skip A/B; distinctly different changes (new behavior, new 成本 curve) require A/B.

硬性拒绝：
- Skipping shadow mode. 拒绝 ： 成本 spikes and length regressions slip past offline eval.
- Gates tighter than 15% 方差. 拒绝 ： false alarms will halt legitimate rollouts.
- 回滚 that 需要 redeploy. 拒绝 ： it is not 一个回滚, it is a damage report.

拒绝规则：
- If the change is safety-关键 (e.g., PII handling change), require explicit additional gate: zero PII leakage in shadow sample before starting 金丝雀.
- If 流量 volume is <100 req/小时, require extended 金丝雀 stages ： otherwise gate noise overwhelms signal.
- If 这个团队 cannot provide 基线指标 for the five 金丝雀 gates, 拒绝 the rollout ： baseline is prerequisite.

输出： a one-page 运行手册 with shadow, 金丝雀, gates, tooling, 回滚, A/B posture. End with 一个回滚 drill requirement: rehearse 回滚 once before first real deploy.
