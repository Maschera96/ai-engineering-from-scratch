# OpenAI Agents SDK：Handoffs、Guardrails、Tracing

> OpenAI Agents SDK 是构建在 Responses API 之上的轻量级多智能体框架。五个基本原语：Agent、Handoff、Guardrail、Session、Tracing。Handoffs 是命名为 `transfer_to_<agent>` 的工具。Guardrails 在输入或输出上触发。Tracing 默认开启。

**类型：** 学习 + 构建
**语言：** Python（标准库）
**前置知识：** Phase 14 · 01（智能体循环）、Phase 14 · 06（工具使用）
**时长：** 约 75 分钟

## 学习目标

- 说出 OpenAI Agents SDK 的五个基本原语。
- 解释 handoffs：为什么把它们建模为工具、模型看到的名称形态是什么、以及上下文如何传递。
- 区分输入 guardrails、输出 guardrails 和工具 guardrails；解释 `run_in_parallel` 与阻塞模式的区别。
- 用标准库实现一个带 handoffs + guardrails + span 风格 tracing 的运行时。

## 问题所在

不能干净地委派的智能体，最终会把所有东西都塞进一个 prompt 里。没有 guardrails 的智能体会泄露 PII、输出违反策略的内容，或者无限循环。OpenAI 的 SDK 把让多智能体工作变得可控的三个基本原语固化了下来。

## 概念

### 五个基本原语

1。**Agent。** LLM + 指令 + 工具 + handoffs。
2。**Handoff。** 委派给另一个智能体。向模型表示为一个命名为 `transfer_to_<agent_name>` 的工具。
3。**Guardrail。** 对输入（仅第一个智能体）、输出（仅最后一个智能体）或工具调用（每个 function tool）进行校验。
4。**Session。** 跨轮次自动维护对话历史。
5。**Tracing。** 为 LLM 生成、工具调用、handoffs、guardrails 内置的 spans。

### 把 handoffs 当作工具

模型在它的工具列表里看到 `transfer_to_billing_agent`。调用它会向运行时发出信号，让它：

1。复制对话上下文（或通过 `nest_handoff_history` bet折叠它）。
2。用目标智能体的指令初始化它。
3。用目标智能体继续这次运行。

这就是产品化后的监督者模式（第 13 课 / 第 28 课）。

### Guardrails

三种风味：

- **输入 guardrails。** 在第一个智能体的输入上运行。在任何 LLM 调用之前拒绝不安全或超出范围的请求。
- **输出 guardrails。** 在最后一个智能体的输出上运行。捕获 PII 泄露、策略违规、格式错误的响应。
- **工具 guardrails。** 按 function tool 运行。校验参数、检查权限、审计执行。

模式：

- **并行**（默认）。Guardrail LLM 与主 LLM 并行运行。尾部延迟更低。如果触发，主 LLM 的工作会被丢弃（浪费 token）。
- **阻塞**（`run_in_parallel=False`）。Guardrail LLM 先运行。如果触发，主调用上不会浪费 token。

触发线（触发wires）会抛出 `InputGuardrailTripwireTriggered` / `OutputGuardrailTripwireTriggered`。

### Tracing

默认开启。每次 LLM 生成、工具调用、handoff 和 guardrail 都会发出一个 span。`OPENAI_AGENTS_DISABLE_TRACING=1` 可以退出。`add_trace_processor(processor)` 会把 spans 分发到你自己的后端，与 OpenAI 的并行。

### Sessions

`Session` 把对话历史存储在某个后端（SQLite、Redis、自定义）。`Runner.run(agent，input，session=session)` 会自动加载并追加。

### 这个模式会在哪里出错

- **Handoff 漂移。** 智能体 A 交给智能体 B，B 又交回给 A。加一个跳数计数器。
- **Guardrail 绕过。** 工具 guardrails 只在 function tools 上触发；内置工具（文件读取器、网页抓取）需要单独的策略。
- **过度 tracing。** spans 里包含敏感内容。与 OTel GenAI 内容捕获规则（第 23 课）配合使用——存储在外部，用 ID 引用。

## 动手构建

`code/main.py` 用标准库实现了 SDK 的形态：

- `Agent`、`FunctionTool`、`Handoff`（作为一个带转移语义的 function tool）。
- 带输入/输出/工具 guardrails、handoff 分派和跳数计数器的 `Runner`。
- 一个简单的 span 发射器，用来展示 trace 的形态。
- 一个分诊智能体，根据用户的查询将其交给计费或支持智能体；在某个输入上 guardrail 会触发。

运行它：

```
python3 code/main.py
```

这个 trace 展示了两次成功的 handoffs、一次输入 guardrail 触发，以及一棵镜像真实 SDK 所发出内容的 span 树。

## 使用它

- **OpenAI Agents SDK** 用于以 OpenAI 为先的产品。
- **Claude Agent SDK**（第 17 课）用于以 Claude 为先的产品。
- **LangGraph**（第 13 课）当你想要显式状态和持久化恢复时。
- **自定义** 当你需要精确控制（语音、多供应商、联邦部署）时。

## 交付它

`outputs/skill-agents-sdk-scaffold.md` 脚手架出一个 Agents SDK 应用，包含一个分诊智能体、handoffs、输入/输出/工具 guardrails、session 存储和一个 trace processor。

## 练习

1。添加一个 handoff 跳数计数器：在 N 次转移后拒绝。追踪其行为。
2。把 `nest_handoff_history` 实现为一个选项——在转移之前将先前的消息折叠成一份摘要。
3。写一个阻塞式输出 guardrail。在会触发它的 prompt 和能通过的 prompt 上比较延迟。
4。把 `add_trace_processor` 接到一个 JSON logger 上。它为每个 span 发出什么形态？
5。阅读 SDK 文档。把你的标准库玩具移植到 `openai-agents-python`。你建模错了什么？

## 关键术语

| 术语 | 人们常说 | 它实际的含义 |
|------|----------------|------------------------|
| Agent | "LLM + 指令" | SDK 中的 Agent 类型；拥有工具和 handoffs |
| Handoff | "转移" | 模型调用以委派给另一个智能体的工具 |
| Guardrail | "策略检查" | 对输入 / 输出 / 工具调用的校验 |
| Tripwire | "Guardrail 触发" | guardrail 拒绝时抛出的异常 |
| Session | "历史存储" | 在多次运行之间持久化的对话记忆 |
| Tracing | "Spans" | 覆盖 LLM + 工具 + handoff + guardrail 的内置可观测性 |
| 阻塞式 guardrail | "顺序检查" | guardrail 先运行；触发时不浪费 token |
| 并行 guardrail | "并发检查" | guardrail 并行运行；延迟更低，触发时浪费 token |

## 延伸阅读

- [OpenAI Agents SDK docs](https://openai.github.io/openai-agents-python/) — 基本原语、handoffs、guardrails、tracing
- [Claude Agent SDK overview](https://platform.claude.com/docs/en/agent-sdk/overview) — Claude 风味的对应物
- [Anthropic，构建ing Effective Agents](https://www.anthropic.com/research/building-effective-agents) — 什么时候才该动用 handoffs
- [OpenTelemetry GenAI semantic conventions](https://opentelemetry.io/docs/规范s/semconv/gen-ai/) — Agents SDK spans 所映射到的标准
