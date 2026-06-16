---
name: gateway-picker-zh
description: 选择an AI 网关 (LiteLLM, Portkey, Kong AI, Cloudflare/Vercel) given scale, 延迟 budget, 合规, ops posture, 和 定价 tolerance.
version: 1.0.0
phase: 17
lesson: 19
tags: [ai-gateway, litellm, portkey, kong, cloudflare, vercel, bifrost, fallback, rate-limit, guardrails]
---

给定RPS (current and projected 12-月), 延迟 budget, 合规 (self-host 必需?), 护栏 need (PII redaction, jailbreak detection, audit), 和 定价 tolerance, produce 一个网关 建议.

产出：

1. 主要网关. Name the tool. 说明理由 with RPS ceiling, overhead, 和 功能 fit.
2. Fallback chain. Three 提供商 in order; OpenAI → Anthropic → 自托管 is canonical. Compute expected availability.
3. 限流 策略. Sliding-window recommended >500 RPS; 词元-bucket acceptable otherwise. Per-tenant tiering.
4. 护栏. Portkey if PII/jailbreak 必需; Kong if need scale + 护栏; LiteLLM if dev tier only.
5. 可观测性 hand-off. Point to Phase 17 · 13 pick; confirm OTel GenAI conventions flow through.
6. 迁移. If moving from app-level integration, staged rollout (1% 金丝雀 on 网关, expand on success).

硬性拒绝：
- LiteLLM at >2000 RPS. 拒绝 ： Kong 基准 shows cascade failures; migrate first.
- Portkey at TTFT P99 < 100 ms SLA. 拒绝 ： 30 ms overhead eats too much of the budget.
- Cloudflare AI 网关 for a regulated on-prem 客户. 拒绝 ： managed-only; no self-host.

拒绝规则：
- If scale ambiguity is large (current 100 RPS, planned 2K+ in 6 months), require 这个迁移 计划 before committing to LiteLLM.
- If 合规 需要 SOC 2 Type II and the chosen 网关 is OSS-only without managed SLA, require 客户's own SOC 2 attestation.
- If 这个团队 has no Kubernetes and picks Kong self-host, 拒绝 ： recommend managed Kong or Portkey managed.

输出： a one-page 决策 搭配 网关, fallback chain, 限流 策略, guardrail posture, 可观测性 flow, 迁移 计划. End with one 指标: 网关 延迟 P99 over last 小时; alert on breach.
