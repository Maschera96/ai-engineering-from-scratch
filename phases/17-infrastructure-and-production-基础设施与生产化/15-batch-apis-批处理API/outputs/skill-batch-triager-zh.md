---
name: batch-triager-zh
description: Triage LLM 工作负载 into interactive / semi-interactive / 批处理 lanes, compute stacked discount (批处理 + cache) 节省, and flag mis-triaged 工作负载.
version: 1.0.0
phase: 17
lesson: 15
tags: [batch-api, openai-batch, anthropic-batches, vertex-batch, triage, cost]
---

给定一个工作负载 (name, user expectation for 延迟, 流量 volume, shared 提示词 structure), produce a triage + 成本 计划.

产出：

1. Lane. Interactive (TTFT-bound, sync), semi-interactive (minutes OK, async 队列), 或 批处理 (by-morning OK, 批处理 API). 说明理由 with 这个具体 user expectation.
2. Current 成本. Compute 每月成本 at current configuration (sync, no cache, etc.).
3. Target 成本. Compute 成本 after recommended config (批处理 + cache or sync + cache). Express as % of current.
4. 迁移 计划. 提供商-具体 steps (pick the one that matches 这个工作负载's 模型, not both):
   - OpenAI: migrate to `/v1/batches`. 提示词缓存 is enabled automatically for eligible 提示词 (≥1024 词元) ： no `cache_control` to set. Optionally pass `prompt_cache_key` for tighter attribution.
   - Anthropic: migrate to Message Batches. Cache reuse 需要 explicit `cache_control` blocks (e.g., `{"type": "ephemeral"}`) on the cacheable 提示词 spans; 批处理 discount stacks with cached-read 定价.
   - Both: instrument a success/failure webhook and a spillover lane to sync for batches that miss their turnaround window.
5. 风险. What if 这个批处理 turnaround is 20 hours at P99? Name the downstream system behavior (email delivery, 队列 spillover to sync).
6. Observable. 指标 that catches mis-triage: 批处理 job completion 延迟 P95; alert if > 12 hours.

硬性拒绝：
- Running an overnight pipeline in sync mode without 批处理 when the user only needs "by morning" 延迟. 拒绝 ： call out 这个~90% leaked spend.
- Promising 批处理 for anything with a sub-15-minute user expectation. 拒绝 ： 批处理 SLA is 24h.
- Ignoring 提示词缓存 on 一个批处理 工作负载 with shared system 提示词. 拒绝 ： the stacked discount is the point.

拒绝规则：
- If 这个工作负载 is marketed as "real-time" but the actual user expectation is minutes, require explicit confirmation before recommending 批处理.
- If 这个工作负载 targets 一个提供商 不带 提示词缓存 in 批处理 (e.g., any custom or 自托管 stack without KV-prefix reuse), note that only 这个批处理 discount applies and recompute without stacked 节省. OpenAI 批处理 caching is automatic; Anthropic 批处理 caching 需要 explicit `cache_control` blocks.
- If 这个工作负载 has strict 延迟 SLA (e.g., P99 < 60s) 拒绝 批处理 outright ： it belongs on a different lane.

输出： a one-page triage with lane, current 成本, target 成本, 迁移 steps, 风险, observable. End with a cadence: re-triage all 工作负载 quarterly as 产品 界面 changes.
