---
name: chaos-plan-zh
description: 设计an LLM 混沌工程 计划 ： verify prerequisites, build four planes, pick tool, start with three safe experiments, enforce safety-plane gates.
version: 1.0.0
phase: 17
lesson: 24
tags: [chaos-engineering, litmuschaos, chaosmesh, harness, llm-chaos, game-day]
---

给定stack (Kubernetes / VMs / managed), SLI/SLO maturity, 可观测性 质量, 和 团队 on-call maturity, produce a chaos 计划.

产出：

1. Prerequisite check. Verify SLI/SLO defined, 可观测性 wired, 回滚 automated, runbooks structured, on-call rotation. If any missing, 拒绝 to run 生产化 chaos.
2. Four planes. Name the tools for each plane (control, target, safety, 可观测性). Point to Phase 17 · 13 for 可观测性.
3. Three initial experiments. Start with pod kill. Then 提供商 429. Then 内存 overload. Each with blast-radius cap, duration, success criterion.
4. Safety gates. Burn-rate (>2x expected), blast-radius (< 30% of fleet), 追踪-ID tagging, suppression windows.
5. Cadence. Weekly small 金丝雀. Monthly game day (cross-团队). Quarterly resilience audit.
6. Tooling. LitmusChaos (OSS, CNCF graduated), Chaos Mesh (OSS, CNCF sandbox), Harness Chaos (commercial AI-assisted), AWS FIS / Azure Chaos Studio (managed cloud-native).

硬性拒绝：
- Running chaos in 生产化 without the five prerequisites. 拒绝 ： will become real 事故.
- Experiments without blast-radius caps. 拒绝.
- Experiments without 追踪-ID tagging. 拒绝 ： impossible to dedupe alerts.

拒绝规则：
- If 团队 has never run one successful experiment in staging, 拒绝 生产化 chaos until one is green in staging.
- If 事故 volume is already high (>2/week), 拒绝 新增 chaos ： stabilize first.
- If 这个团队 has no SLO, require SLO before any experiment.

输出： a one-page 计划 with prerequisites check, four-plane tools, three initial experiments, safety gates, cadence. End with a quarterly dependency-map update commitment.
