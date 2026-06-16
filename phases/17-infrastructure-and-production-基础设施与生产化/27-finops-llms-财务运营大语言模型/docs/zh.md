# 大模型 FinOps：单位经济学与多租户归因

> Traditional FinOps breaks on LLM spend. Costs are 词元-transactions, not resource-uptime. 标签 don't map ： an API call is a transaction, not an asset. Engineering decisions (提示词 design, context window, 输出 length) are financial decisions. The 2026 playbook has three attribution dimensions to instrument on day one: per-user (`user_id`) for seat 定价 and expansion, per-task (`task_id` + `route`) for 产品 界面 成本 and prioritization, per-tenant (`tenant_id`) for 单位经济学 and renewal. Four 词元 layers ： 提示词, tool, 内存, response ： one bucket hides spend. Enforcement ladder for 多租户 products: 限流 per tenant (2-3x expected peak, clear 429 + retry-after); daily spend cap (1.5-3x contracted ceiling; triggers rate tightening + alert); kill switches on spend z-score > 4 (auto-pause + page on-call). Attribution patterns: tag-and-aggregate, telemetry-joiner (追踪-ID → billing; highest accuracy), sampling-and-extrapolation, 模型-based allocation, event-sourced, real-time streaming. Unit 指标: 成本 per resolved query, 成本 per generated artifact ： not $/M 词元. Retroactive tagging always misses; instrument at 请求 creation.

**Type:** Learn
**Languages:** Python（标准库， 玩具 cost-attribution模拟器 with kill switch)
**Prerequisites:** Phase 17 · 13 (Observability), Phase 17 · 14 (Caching)
**Time:** ~60 分钟

## 学习目标

- 解释 why traditional FinOps (标签 + tiers) breaks on LLM spend and name the three new attribution dimensions.
- Enumerate the four 词元 layers (提示词, tool, 内存, response) and why single-bucket billing hides 成本.
- Design an enforcement ladder (rate → spend cap → kill switch) for 一个多租户 产品.
- Pick a unit 指标 (成本 per resolved query / artifact) instead of $/M 词元.

## 问题

Your bill says $40,000. You don't know:
- Which tenant spent it.
- Which 产品 功能 drove it.
- Whether any individual user was abusive.
- Whether 提示词 bloat, tool calls, 或 内存 amplification was 这个元凶.

Tag-and-aggregate on 提供商-side works for cloud resources (EC2, S3) 哪里 标签 propagate to line items. LLM API calls do not auto-tag ： you have to stamp user/task/tenant at the call site and carry through. Retroactive attribution always misses edge cases.

## 概念

### 三个归因维度

**Per-user** (`user_id`): who is costing what. Drives seat 定价, expansion conversations, identifies power users.

**Per-task** (`task_id` + `route`): which 产品 界面 costs what. Drives 功能 prioritization, kill-expensive-功能 decisions.

**Per-tenant** (`tenant_id`): which 客户 is profitable. Drives 单位经济学, renewal 定价, tier thresholds.

Instrument all three at call site on day one. Retroactive is always worse.

### 四个词元层

| Layer | Example | Typical % of total |
|-------|---------|---------------------|
| 提示词 | system + user 输入 | 40-60% |
| Tool | tool-call results fed back | 20-40% (agent 工作负载) |
| 内存 | prior conversation / retrieved docs | 10-30% |
| Response | 模型 输出 | 10-30% |

Bucketing all four together makes optimization blind. Break them out in your attribution schema.

### 执行阶梯

1. **Rate limit** per tenant. 2-3x expected peak. Return 429 with `Retry-After`. Tenant sees friction; no surprise bill.

2. **Daily spend cap** per tenant. 1.5-3x contracted ceiling. Trigger: tighten rate limit + alert 客户-success.

3. **Kill switch** on spend z-score > 4 relative to tenant baseline. Auto-pause tenant; page on-call; escalate to ops + CS.

### 归因模式

- **Tag-and-aggregate**: stamp metadata 头; aggregate later. 简单; rough.
- **Telemetry joiner**: join 追踪 to billing via 追踪 IDs. Highest accuracy. What mature 团队 do.
- **Sampling + extrapolation**: sample 5-10%, multiply. 成本-effective for rough spend; misses tails.
- **模型-based allocation**: regression to infer 成本 driver. For legacy data without 标签.
- **Event-sourced**: 成本 as events in a stream (Kafka / Kinesis). Real-time.
- **Real-time streaming**: dashboard updates sub-second.

### 每 X 成本是单位指标

$/M 词元 is vendor speak. 产品 指标:

- 成本 per resolved support ticket.
- 成本 per generated article.
- 成本 per successful agent task.
- 成本 per user-session-minute.

Tie 成本 to 一个产品 outcome. Otherwise optimization is unanchored.

### 成本归因 追踪 shape

```
trace_id: abc123
  user_id: u_42
  tenant_id: t_7
  task_id: task_classify_doc
  route: model_haiku
  layers:
    prompt_tokens: 1800
    tool_tokens: 600
    memory_tokens: 400
    response_tokens: 150
  cost_usd: 0.0135
  cached_input: true
  batch: false
```

Emit on every call. Store in data lake. Aggregate per dimension. Phase 17 · 13 可观测性 stack is where this lives.

### The compounded-节省 stack

Stack: cache + 批处理 + 路由 + 网关. With all four:
- Cache L2 (Phase 17 · 14): ~10x 更便宜 输入.
- 批处理 (Phase 17 · 15): 50% off.
- 路由 to cheap 模型 (Phase 17 · 16): 60% 成本 reduction.
- 网关 efficiency (Phase 17 · 19): redundancy + retries.

Best-case stacked: ~5-10% of naive baseline. Most 团队 have 2-3 levers engaged; few stack all four.

### 你应该记住的数字

- Attribution dimensions: per-user, per-task, per-tenant.
- Four 词元 layers: 提示词, tool, 内存, response.
- Kill switch: spend z-score > 4.
- Unit 指标: 成本 per resolved query, not $/M 词元.
- Stacked optimizations: ~5-10% of baseline possible.

## 使用它

`code/main.py` simulates 一个多租户 LLM service with the three-tier enforcement ladder. Injects an abusive tenant and demonstrates the kill switch firing.

## 交付它

This lesson 产出 `outputs/skill-finops-plan.md`. 给定产品 and scale, designs the attribution schema and enforcement ladder.

## 练习

1. Run `code/main.py`. At what z-score does the kill switch fire? How do you pick the threshold?
2. Design a per-tenant, per-task 成本 dashboard. What are the 5 views you build first?
3. Your largest tenant is unit-economics-negative. Propose three interventions ordered by 客户 impact.
4. Compute 成本 per resolved ticket for a support 产品: 3M 词元/ticket, ~800 tickets/day, GPT-5 cached rate.
5. Argue whether retroactive tagging can ever work. When is it acceptable?

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| Per-user attribution | "user-level 成本" | `user_id` stamped on every call |
| Per-task attribution | "功能 成本" | `task_id` + `route` identify 产品 界面 |
| Per-tenant attribution | "客户 成本" | `tenant_id`; drives 单位经济学 |
| Four 词元 layers | "成本 layers" | 提示词 + tool + 内存 + response |
| Rate limit | "429 guard" | Per-tenant ceiling enforced at 网关 |
| Daily spend cap | "daily ceiling" | Tenant-scoped budget with alert |
| Kill switch | "auto-pause" | Spend z-score > 4 triggers auto-suspension |
| 成本 per resolved | "产品 unit 指标" | 成本 tied to 产品 outcome, not 词元 |
| Telemetry joiner | "追踪-to-billing" | Highest-accuracy attribution 模式 |
| Stacked optimization | "cache+批处理+路由+网关" | Compounding 节省 to ~5-10% 基线|

## 延伸阅读

- [FinOps Foundation — FinOps for AI Overview](https://www.finops.org/wg/finops-for-ai-overview/)
- [FinOps School — Cost per Unit 2026 Guide](https://finopsschool.com/blog/cost-per-unit/)
- [Digital Applied — LLM Agent Cost Attribution 2026](https://www.digitalapplied.com/blog/llm-agent-cost-attribution-guide-production-2026)
- [PointFive — Managed LLMs in Azure OpenAI](https://www.pointfive.co/blog/finops-for-ai-economics-of-managed-llms-in-azure-open-ai)
