---
name: observability-stack-zh
description: 选择an LLM 可观测性 stack (development 平台 + 网关 + optional scale layer) given stack, scale, budget, and license posture, and define the OpenTelemetry GenAI attribute set.
version: 1.0.0
phase: 17
lesson: 13
tags: [observability, langfuse, langsmith, phoenix, arize, helicone, opik, opentelemetry, genai-conventions]
---

给定stack (LangChain / DSPy / raw SDK), scale (追踪/day), budget, license posture (MIT-only 对比 commercial OK), and self-host requirement, produce 一个可观测性 计划.

产出：

1. Development 平台 choice. Langfuse (OSS), LangSmith (LangChain-first commercial), Opik (Comet OSS), or none. 说明理由 with stack and license.
2. 网关/telemetry choice. Helicone (proxy + 网关), SigNoz (full APM), OpenLLMetry (pure OTel). If already using an AI 网关 (Phase 17 · 19), name the integration.
3. Scale/lake layer. Optional; Arize AX or raw Iceberg for long-term analytics, Phoenix for RAG drift.
4. OTel GenAI conventions. Specify the minimum attribute set: `gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.request.temperature`, `gen_ai.response.finish_reasons`, plus org-具体(tenant_id, user_id, task).
5. Sampling 策略. 100% errors, 100% high-成本 (>$0.10/call), N% success sampling rate. Raw-retention window (14d / 30d / 90d). Aggregates retained longer.
6. Alerting. Five 指标 that must have alerts: error rate, P99 TTFT, 成本/请求, 提示词-cache hit rate, refusal rate.

硬性拒绝：
- Instrumenting inside framework-具体 SDK without an OTel fallback. 拒绝 ： framework lock-in.
- Keeping 100% of 追踪 at Datadog-class 定价 >$500/mo for a non-regulated 工作负载. 拒绝 ： recommend sampling.
- Ignoring OpenTelemetry GenAI conventions. 拒绝 ： 2026 interop 需要 them.

拒绝规则：
- If 追踪/day > 5M and 这个团队 insists on full Datadog retention, 拒绝 without 一个成本 forecast.
- If 这个团队 is MIT-only and picks LangSmith, 拒绝 ： Langfuse is the MIT equivalent.
- If 这个团队 has no AI 网关 and picks Helicone as 网关 和 可观测性, accept ： the proxy doubles as 网关 up to ~500 RPS (Phase 17 · 19 covers 网关 scale).

输出： a one-page 计划 naming dev 平台, 网关, scale layer (if any), OTel attribute set, sampling rule, five alerts. End with the single 指标 that signals stack drift: percentage of LLM calls with complete OTel GenAI attributes over last 7 days.
