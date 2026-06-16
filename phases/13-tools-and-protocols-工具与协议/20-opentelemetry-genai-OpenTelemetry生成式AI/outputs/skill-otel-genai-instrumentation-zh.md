---
name: otel-genai-instrumentation-zh
description: 为智能体代码库生成一份端到端发射 OTel GenAI span 的插桩方案。
version: 1.0.0
phase: 13
lesson: 19
tags: [otel, observability, gen-ai, tracing]
---

给定一个智能体代码库（LLM 调用、工具分发、MCP 客户端、子智能体），生成一份 OTel GenAI 插桩方案。

产出：

1. Span 层级。根 span `agent.invoke_agent`（INTERNAL）及其子 span：`llm.chat`（CLIENT）、`tool.execute`（INTERNAL）、`mcp.call`（CLIENT）、`subagent.invoke`（INTERNAL）。
2. 每个 span 的属性检查清单。`gen_ai.operation.name`、`gen_ai.provider.name`、`gen_ai.request.model`、`gen_ai.response.model`、`gen_ai.usage.*`、`gen_ai.tool.name`、`gen_ai.agent.name`。
3. 传播规则。在每次远程调用上注入 W3C traceparent；对于 MCP stdio，使用 `_meta.traceparent` 作为过渡字段。
4. 内容捕获策略。默认关闭；记录哪个环境变量可启用它；指明 PII 风险。
5. 导出器选择。Jaeger / Tempo / Langfuse / Phoenix / Datadog / Honeycomb；以 OTLP 作为传输协议。

硬性拒绝：
- 任何缺少跨 MCP 或子智能体边界的 trace 传播的方案。
- 任何默认开启内容捕获的方案。会泄露提示词与 PII。
- 任何发射不带 `gen_ai.` 或明确厂商前缀的任意自定义属性的方案。

拒绝规则：
- 如果代码库使用的框架自带 OTel 自动插桩（Pydantic AI、LangGraph、AgentOps），优先推荐该框架的 hook。
- 如果导出器后端是本地部署（on-prem）且团队没有 SRE 支持，推荐使用托管后端。
- 如果用户要求在生产环境中捕获内容用于调试，在没有类型化的同意策略与 PII 脱敏管道的情况下予以拒绝。

输出：一份一页式方案，包含 span 层级、每个 span 的属性检查清单、传播规则、内容捕获策略以及导出器选择。结尾给出最该告警的指标（通常是 p95 `gen_ai.client.operation.duration`）。
