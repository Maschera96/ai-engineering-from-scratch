# OpenTelemetry GenAI 语义约定

> OpenTelemetry 的 GenAI SIG（于 2024 年 4 月启动）定义了智能体遥测的标准模式。span 名称、属性和内容捕获规则在各厂商之间趋于统一，因此智能体追踪在 Datadog、Grafana、Jaeger 和 Honeycomb 中含义一致。

**类型：** 学习 + 构建
**语言：** Python（标准库）
**前置条件：** Phase 14 · 13（LangGraph），Phase 14 · 24（可观测性平台）
**时长：** 约 60 分钟

## 学习目标

- 说出 GenAI span 的类别：model/client、agent、tool。
- 区分 `invoke_agent` 的 CLIENT span 与 INTERNAL span，以及各自适用的场景。
- 列出顶层 GenAI 属性：provider 名称、request 模型、数据源 ID。
- 解释内容捕获契约：选择加入（opt-in）、`OTEL_SEMCONV_STABILITY_OPT_IN`、外部引用建议。

## 问题所在

每家厂商都自创自己的 span 名称。运维团队最终不得不为每个框架分别构建仪表盘。OpenTelemetry 的 GenAI SIG 通过定义整个生态系统共同遵循的单一标准来解决这个问题。

## 核心概念

### span 类别

1. **Model / client span。** 覆盖原始的 LLM 调用。由 provider SDK（Anthropic、OpenAI、Bedrock）和框架的模型适配器发出。
2. **Agent span。** `create_agent`（构造智能体时）和 `invoke_agent`（智能体运行时）。
3. **Tool span。** 每次工具调用一个；通过父子关系连接到 agent span。

### agent span 命名

- span 名称：若已命名，则为 `invoke_agent {gen_ai.agent.name}`；否则回退为 `invoke_agent`。
- span 类型：
  - **CLIENT** —— 用于远程智能体服务（OpenAI Assistants API、Bedrock Agents）。
  - **INTERNAL** —— 用于进程内的智能体框架（LangChain、CrewAI、本地 ReAct）。

### 关键属性

- `gen_ai.provider.name` —— `anthropic`、`openai`、`aws.bedrock`、`google.vertex`。
- `gen_ai.request.model` —— 模型 ID。
- `gen_ai.response.model` —— 解析后的模型（因路由可能与 request 不同）。
- `gen_ai.agent.name` —— 智能体标识符。
- `gen_ai.operation.name` —— `chat`、`completion`、`invoke_agent`、`tool_call`。
- `gen_ai.data_source.id` —— 用于 RAG：查询了哪个语料库或存储。

针对 Anthropic、Azure AI Inference、AWS Bedrock、OpenAI 还存在特定技术的约定。

### 内容捕获

默认规则：插桩 SHOULD NOT 在默认情况下捕获输入/输出。捕获是选择加入的，通过以下属性实现：

- `gen_ai.system_instructions`
- `gen_ai.input.messages`
- `gen_ai.output.messages`

推荐的生产模式：将内容存储在外部（S3、你的日志存储），在 span 上记录引用（指针 ID，而非正文）。这正是第 27 课的内容投毒防御接入到可观测性中的体现。

### 稳定性

截至 2026 年 3 月，大多数约定仍处于实验阶段。通过以下方式选择加入稳定预览版：

```
OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental
```

Datadog v1.37+ 将 GenAI 属性原生映射到其 LLM Observability 模式中。其他后端（Grafana、Honeycomb、Jaeger）支持原始属性。

### 这种模式哪里会出错

- **在 span 中捕获完整的提示词。** 运维人员可读取的追踪中包含 PII、密钥、客户数据。请存储在外部。
- **缺少 `gen_ai.provider.name`。** 当归因缺失时，多 provider 仪表盘会失效。
- **没有父链接的 span。** 孤立的 tool span。务必传播上下文。
- **未设置稳定性选择加入。** 你的属性可能在后端升级时被重命名。

## 动手构建

`code/main.py` 实现了一个符合 GenAI 约定的标准库 span 发射器：

- 带有 GenAI 属性模式的 `Span`。
- 带有 `start_span`、嵌套上下文的 `Tracer`。
- 一次脚本化的智能体运行，发出：`create_agent`、`invoke_agent`（INTERNAL）、每个工具的 span、用于 LLM 调用的 `chat` span。
- 一种内容捕获模式，将提示词存储在外部并在 span 上记录 ID。

运行它：

```
python3 code/main.py
```

输出：一棵包含所有必需 GenAI 属性的 span 树，以及一个展示选择加入内容引用的“外部存储”。

## 使用它

- **Datadog LLM Observability**（v1.37+）原生映射属性。
- **Langfuse / Phoenix / Opik**（第 24 课）—— 为整个生态系统自动插桩。
- **Jaeger / Honeycomb / Grafana Tempo** —— 原始 OTel 追踪；基于 GenAI 属性构建仪表盘。
- **自托管** —— 运行带有 GenAI 处理器的 OTel Collector。

## 交付它

`outputs/skill-otel-genai.md` 将 OTel GenAI span 接入到现有智能体中，附带内容捕获默认值和外部引用存储。

## 练习

1. 用 `invoke_agent`（INTERNAL）+ 每个工具的 span 为你的第 01 课 ReAct 循环插桩。发送到一个 Jaeger 实例。
2. 以“仅引用”模式添加内容捕获：提示词存入 SQLite，span 属性只携带行 ID。
3. 阅读 `gen_ai.data_source.id` 的规范。将其接入你第 09 课的 Mem0 搜索。
4. 设置 `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`，并验证你的属性不会被 collector 重命名。
5. 构建一个仪表盘：仅凭 GenAI 属性，分析“哪些工具错误与哪些模型相关”。

## 关键术语

| 术语 | 大家怎么说 | 实际含义 |
|------|----------------|------------------------|
| GenAI SIG | “OpenTelemetry GenAI 组” | 定义该模式的 OTel 工作组 |
| invoke_agent | “Agent span” | 表示一次智能体运行的 span 名称 |
| CLIENT span | “远程调用” | 用于调用远程智能体服务的 span |
| INTERNAL span | “进程内” | 用于进程内智能体运行的 span |
| gen_ai.provider.name | “Provider” | anthropic / openai / aws.bedrock / google.vertex |
| gen_ai.data_source.id | “RAG 来源” | 检索命中了哪个语料库/存储 |
| Content capture | “提示词日志” | 选择加入的消息捕获；在生产中存储于外部 |
| Stability opt-in | “预览模式” | 用于固定实验性约定的环境变量 |

## 延伸阅读

- [OpenTelemetry GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) —— 规范
- [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/) —— 默认带有 GenAI span
- [AutoGen v0.4 (Microsoft Research)](https://www.microsoft.com/en-us/research/articles/autogen-v0-4-reimagining-the-foundation-of-agentic-ai-for-scale-extensibility-and-robustness/) —— 内置 OTel span
- [Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview) —— W3C 追踪上下文传播
