---
name: finops-plan-zh
description: 设计an LLM FinOps program ： attribution schema (user/task/tenant + four 词元 layers), three-tier enforcement ladder, and unit 指标 (成本 per resolved / artifact).
version: 1.0.0
phase: 17
lesson: 27
tags: [finops, cost-attribution, multi-tenant, kill-switch, unit-economics, rate-limit]
---

给定产品 界面, tenant tiers, monthly spend, and current attribution state, produce a FinOps 计划.

产出：

1. Attribution schema. `user_id`, `task_id`, `route`, `tenant_id` stamped at call site. Four 词元-layer counts (提示词 / tool / 内存 / response). Telemetry-joiner 模式 preferred.
2. Unit 指标. Define 这个产品 outcome 指标 ： 成本 per resolved ticket, 成本 per artifact, 成本 per agent task, 成本 per session. Tie to billing 模型.
3. Enforcement ladder. Rate limit per tenant (2-3x peak), daily spend cap (1.5-3x contract), kill switch on z-score > 4.
4. Dashboard. Top 5 views: per-tenant spend today, per-task 成本-per-outcome, per-user distribution, cache hit rate impact, 模型 路由 split.
5. Stacked optimization audit. Check cache (Phase 17 · 14), 批处理 (Phase 17 · 15), 路由 (Phase 17 · 16), 网关 (Phase 17 · 19) are all engaged. Flag missing levers.
6. Review cadence. Weekly: top spenders + anomalies. Monthly: per-tenant unit-economics. Quarterly: re-triage 工作负载 into interactive/semi/批处理.

硬性拒绝：
- Shipping without attribution at call site. 拒绝 ： retroactive tagging loses ~10-30% of spend.
- Single-bucket billing. 拒绝 ： require four 词元-layer breakdown.
- Kill switch with no z-score basis. 拒绝 ： require baseline statistics before arming.

拒绝规则：
- If 这个产品 has < 10 tenants, 拒绝 full 多租户 enforcement ： require basic per-tenant attribution first.
- If 成本/outcome is undefined, 拒绝 the dashboard ： pick a unit 指标 first.
- If any single tenant is > 40% of total spend, require dedicated unit-economics review before the 计划 ships.

输出： a one-page 计划 with attribution schema, unit 指标, enforcement ladder, dashboard, stacked optimization audit, review cadence. End with the single alert: daily spend 对比 projection; page when delta > 20%.
