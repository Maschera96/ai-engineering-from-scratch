# OpenTelemetry GenAI — 端到端追踪工具调用

> 一个智能体调用了五个工具、三个 MCP server 和两个子智能体。你需要一条贯穿全部环节的 trace。OpenTelemetry GenAI 语义约定（属性在 v1.37 及以上版本中已稳定）是 2026 年的标准，被 Datadog、Langfuse、Arize Phoenix、OpenLLMetry 和 AgentOps 原生支持。本课会列出必需的属性，逐层讲解 span 层级（agent → LLM → tool），并提供一个标准库实现的 span 发射器，你可以将它接入任何 OTel exporter。

**类型：** 构建
**语言：** Python（标准库，OTel span 发射器）
**前置条件：** 阶段 13 · 07（MCP server）、阶段 13 · 08（MCP client）
**时长：** 约 75 分钟

## 学习目标

- 列出一个 LLM span 和一个工具执行 span 所需的 OTel GenAI 属性。
- 构建一个覆盖智能体循环、LLM 调用、工具调用和 MCP client 派发的 trace 层级。
- 决定哪些内容需要捕获（按需开启）、哪些需要脱敏（默认行为）。
- 在不重写工具代码的前提下，将 span 发射到本地 collector（Jaeger、Langfuse）。

## 问题

一次 2026 年 2 月的调试：用户报告"我的智能体有时要 30 秒才响应，有时只要 3 秒"。没有 trace。日志显示了 LLM 调用，但没有显示工具派发，没有 MCP server 往返，也没有子智能体。你只能靠猜。最终你发现：某个 MCP server 偶尔会在冷启动时挂起。

没有端到端追踪，你根本找不到这个问题。OTel GenAI 解决了它。

这些约定在 2025-2026 年由 OpenTelemetry 语义约定工作组确定下来。它们定义了稳定的属性名称，让 Datadog、Langfuse、Phoenix、OpenLLMetry 和 AgentOps 都能解析同样的 span。一次插桩，发往任意后端。

## 概念

### Span 层级

```
agent.invoke_agent  (top, INTERNAL span)
 ├── llm.chat       (CLIENT span)
 ├── tool.execute   (INTERNAL)
 │    └── mcp.call  (CLIENT span)
 ├── llm.chat       (CLIENT span)
 └── subagent.invoke (INTERNAL)
```

整个结构嵌套在一个 trace id 下。span id 链接起父子关系。

### 必需的属性

依据 2025-2026 semconv：

- `gen_ai.operation.name` — `"chat"`、`"text_completion"`、`"embeddings"`、`"execute_tool"`、`"invoke_agent"`。
- `gen_ai.provider.name` — `"openai"`、`"anthropic"`、`"google"`、`"azure_openai"`。
- `gen_ai.request.model` — 请求的模型字符串（例如 `"gpt-4o-2024-08-06"`）。
- `gen_ai.response.model` — 实际提供服务的模型。
- `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens`。
- `gen_ai.response.id` — 用于关联的 provider 响应 id。

对于工具 span：

- `gen_ai.tool.name` — 工具标识符。
- `gen_ai.tool.call.id` — 具体的调用 id。
- `gen_ai.tool.description` — 工具描述（可选）。

对于智能体 span：

- `gen_ai.agent.name` / `gen_ai.agent.id` / `gen_ai.agent.description`。

### Span 类型

- 跨进程边界的调用（LLM provider、MCP server）使用 `SpanKind.CLIENT`。
- 智能体自身的循环步骤和工具执行使用 `SpanKind.INTERNAL`。

### 按需的内容捕获

默认情况下，span 携带指标和时序，而不携带 prompt 或补全内容。大型负载和 PII 默认关闭。设置 `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental` 和特定的内容捕获环境变量来包含内容。在生产环境启用前请仔细审查。

### Span 上的事件

token 级别的事件可以作为 span 事件添加：

- `gen_ai.content.prompt` — 输入消息。
- `gen_ai.content.completion` — 输出消息。
- `gen_ai.content.tool_call` — 记录下来的工具调用。

事件在一个 span 内按时间排序，便于详细回放。

### Exporter

OTel span 可导出到：

- **Jaeger / Tempo。** 开源，本地部署。
- **Langfuse。** 专注 LLM 可观测性；可视化 token 使用量。
- **Arize Phoenix。** 评估 + 追踪相结合。
- **Datadog。** 商业产品；原生解析 `gen_ai.*` 属性。
- **Honeycomb。** 面向列存储；查询友好。

它们都使用线缆格式 OTLP。你的代码无需关心。

### 跨 MCP 的传播

当一个 MCP client 调用 server 时，将 W3C traceparent 头注入到请求中。Streamable HTTP 支持标准头。Stdio 本身不携带 HTTP 头；规范的 2026 路线图讨论了在 JSON-RPC 调用上增加一个 `_meta.traceparent` 字段。

在该功能落地之前：手动在每个请求的 `_meta` 中包含 traceparent。server 记录 trace id。

### 指标

除 span 之外，GenAI semconv 还定义了指标：

- `gen_ai.client.token.usage` — 直方图。
- `gen_ai.client.operation.duration` — 直方图。
- `gen_ai.tool.execution.duration` — 直方图。

对于不需要逐次调用细节的仪表盘，使用这些指标。

### AgentOps 层

AgentOps（成立于 2024 年）专注于 GenAI 可观测性。它包装了流行的框架（LangGraph、Pydantic AI、CrewAI）以自动发射 OTel span。如果你的技术栈使用受支持的框架，它会很有用；否则使用手动插桩。

## 用起来

`code/main.py` 为一个智能体发射 OTel 形状的 span 到标准输出（以类 OTLP-JSON 格式），该智能体会调用一个 LLM、派发两个工具，并进行一次 MCP 往返。没有真实的 exporter——本课聚焦于 span 的形状和属性集。把输出粘贴到兼容 OTLP 的查看器，或者直接阅读它。

需要关注的点：

- trace id 在所有 span 之间共享。
- 父子链接通过 `parentSpanId` 编码。
- 必需的 `gen_ai.*` 属性已填充。
- 内容捕获默认关闭；有一个场景通过环境变量将其打开。

## 交付

本课产出 `outputs/skill-otel-genai-instrumentation.md`。给定一个智能体代码库，该技能会产出一份插桩方案：在哪里添加 span、需要填充哪些属性、以及面向哪些 exporter。

## 练习

1. 运行 `code/main.py`。数一数有多少个 span，并辨认哪些是 CLIENT、哪些是 INTERNAL。

2. 打开内容捕获（环境变量），确认 `gen_ai.content.prompt` 和 `gen_ai.content.completion` 事件出现。注意它对 PII 的影响。

3. 添加工具执行指标 `gen_ai.tool.execution.duration`，并为每次调用发射一个直方图样本。

4. 将一个 traceparent 从父智能体 span 传播到一个 MCP 请求的 `_meta.traceparent` 字段。验证 MCP server 会看到同样的 trace id。

5. 阅读 OTel GenAI semconv 规范。找出 semconv 中列出但本课代码并未发射的一个属性。把它加上。

## 关键术语

| 术语 | 大家怎么说 | 实际含义 |
|------|----------------|------------------------|
| OTel | "OpenTelemetry" | 用于 trace、指标、日志的开放标准 |
| GenAI semconv | "GenAI 语义约定" | 用于 LLM / 工具 / 智能体 span 的稳定属性名称 |
| `gen_ai.*` | "属性命名空间" | 所有 GenAI 属性共享这个前缀 |
| Span | "计时操作" | 一个有起点、终点和属性的工作单元 |
| Trace | "跨 span 的谱系" | 共享一个 trace id 的 span 树 |
| SpanKind | "CLIENT / SERVER / INTERNAL" | 关于 span 方向的提示 |
| OTLP | "OpenTelemetry Line Protocol" | exporter 使用的线缆格式 |
| 按需内容 | "prompt / 补全捕获" | 默认关闭；通过环境变量启用 |
| traceparent | "W3C 头" | 跨服务传播 trace 上下文 |
| Exporter | "后端专用发送器" | 将 span 发送到 Jaeger / Datadog / 等的组件 |

## 延伸阅读

- [OpenTelemetry — GenAI semconv](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — GenAI span、指标和事件的权威约定
- [OpenTelemetry — GenAI spans](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/) — LLM 和工具执行 span 的属性列表
- [OpenTelemetry — GenAI agent spans](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/) — 智能体级别的 `invoke_agent` span
- [open-telemetry/semantic-conventions — GenAI spans](https://github.com/open-telemetry/semantic-conventions/blob/main/docs/gen-ai/gen-ai-spans.md) — GitHub 托管的事实来源
- [Datadog — LLM OTel semantic convention](https://www.datadoghq.com/blog/llm-otel-semantic-convention/) — 生产集成实操指南
