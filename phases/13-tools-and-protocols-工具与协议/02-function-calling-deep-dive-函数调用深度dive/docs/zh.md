# 函数调用深度剖析 —— OpenAI、Anthropic、Gemini

> 三家前沿厂商在 2024 年趋同于同一套工具调用循环，随后在其他方面各奔东西。OpenAI 使用 `tools` 和 `tool_calls`。Anthropic 使用 `tool_use` 和 `tool_result` 块。Gemini 使用 `functionDeclarations` 和唯一 id 关联。本课把三者并排对比，让在一家厂商上交付的代码移植到其他厂商时不会崩。

**类型：** 构建
**语言：** Python（stdlib，模式翻译器）
**前置：** Phase 13 · 01（工具接口）
**时长：** 约 75 分钟

## 学习目标

- 说出 OpenAI、Anthropic 和 Gemini 函数调用载荷的三处形态差异（声明、调用、结果）。
- 把一个工具声明翻译成全部三种厂商格式，并预测严格模式约束会在哪里不同。
- 在每家厂商中使用 `tool_choice` 来强制、禁止或自动挑选工具调用。
- 了解每家厂商的硬性上限（工具数量、模式深度、参数长度），以及违反上限时各自发出的错误特征。

## 问题

函数调用请求的形态因厂商而异。下面是 2026 年生产栈中的三个具体示例：

**OpenAI Chat Completions / Responses API。** 你传入 `tools: [{type: "function", function: {name, description, parameters, strict}}]`。模型的响应包含 `choices[0].message.tool_calls: [{id, type: "function", function: {name, arguments}}]`，其中 `arguments` 是你必须解析的 JSON 字符串。严格模式（`strict: true`）通过受约束的解码强制模式合规。

**Anthropic Messages API。** 你传入 `tools: [{name, description, input_schema}]`。响应回来的形式为 `content: [{type: "text"}, {type: "tool_use", id, name, input}]`。`input` 已经解析好（是一个对象，而非字符串）。你用一条新的 `user` 消息回复，其中包含一个 `{type: "tool_result", tool_use_id, content}` 块。

**Google Gemini API。** 你传入 `tools: [{functionDeclarations: [{name, description, parameters}]}]`（嵌套在 `functionDeclarations` 之下）。响应到达的形式为 `candidates[0].content.parts: [{functionCall: {name, args, id}}]`，其中 `id` 在 Gemini 3 及以上版本是唯一的，用于并行调用关联。你用 `{functionResponse: {name, id, response}}` 回复。

同一个循环。不同的字段名、不同的嵌套、不同的字符串 vs 对象约定、不同的关联机制。一个在 OpenAI 上写了天气智能体的团队，要付出两天的移植成本到 Anthropic，再付一天到 Gemini，仅仅是为了这些管道接线。

本课构建一个翻译器，把三种格式统一成一份规范的工具声明，并在边缘做路由。Phase 13 · 17 把同一模式推广成一个 LLM 网关。

## 概念

### 共同结构

每家厂商都需要五样东西：

1. **工具列表。** 每个工具的名称、描述和输入模式。
2. **工具选择。** 强制某个特定工具、禁止工具，或让模型决定。
3. **调用发出。** 命名工具和参数的结构化输出。
4. **调用 id。** 把响应关联到正确的调用（对并行很重要）。
5. **结果注入。** 把结果绑回调用的一条消息或一个块。

### 形态差异，逐字段对比

| 方面 | OpenAI | Anthropic | Gemini |
|--------|--------|-----------|--------|
| 声明外壳 | `{type: "function", function: {...}}` | `{name, description, input_schema}` | `{functionDeclarations: [{...}]}` |
| 模式字段 | `parameters` | `input_schema` | `parameters` |
| 响应容器 | assistant 消息上的 `tool_calls[]` | 类型为 `tool_use` 的 `content[]` | 类型为 `functionCall` 的 `parts[]` |
| 参数类型 | 字符串化的 JSON | 已解析的对象 | 已解析的对象 |
| id 格式 | `call_...`（OpenAI 生成） | `toolu_...`（Anthropic） | UUID（Gemini 3+） |
| 结果块 | role `tool`，`tool_call_id` | 带 `tool_result` 的 `user`，`tool_use_id` | 带匹配 `id` 的 `functionResponse` |
| 强制某个工具 | `tool_choice: {type: "function", function: {name}}` | `tool_choice: {type: "tool", name}` | `tool_config: {function_calling_config: {mode: "ANY"}}` |
| 禁止工具 | `tool_choice: "none"` | `tool_choice: {type: "none"}` | `mode: "NONE"` |
| 严格模式 | `strict: true` | 模式即模式（始终强制） | 请求级别的 `responseSchema` |

### 你真正会撞上的上限

- **OpenAI。** 每个请求 128 个工具。模式深度 5。参数字符串 <= 8192 字节。严格模式要求没有 `$ref`，没有带重叠的 `oneOf`/`anyOf`/`allOf`，每个属性都列在 `required` 中。
- **Anthropic。** 每个请求 64 个工具。模式深度实际上无界，但实用上限为 10。没有严格模式标志；模式是一份契约，模型倾向于遵从。
- **Gemini。** 每个请求 64 个函数。模式类型是 OpenAPI 3.0 的子集（与 JSON Schema 2020-12 略有偏离）。自 Gemini 3 起并行调用具有唯一 id。

### `tool_choice` 行为

所有人都支持的三种模式，命名各异。

- **Auto。** 模型挑选工具或文本。默认。
- **Required / Any。** 模型必须至少调用一个工具。
- **None。** 模型不得调用工具。

外加每家厂商各自独有的一种模式：

- **OpenAI。** 按名称强制某个特定工具。
- **Anthropic。** 按名称强制某个特定工具；`disable_parallel_tool_use` 标志区分单次与多次。
- **Gemini。** `mode: "VALIDATED"` 无论模型意图如何，都让每个响应经过一个模式校验器。

### 并行调用

OpenAI 的 `parallel_tool_calls: true`（默认）在一条 assistant 消息中发出多个调用。你全部运行它们，并用一条批量的 tool 角色消息回复，其中每个 `tool_call_id` 对应一个条目。Anthropic 历史上是单次调用；`disable_parallel_tool_use: false`（自 Claude 3.5 起为默认）启用多次调用。Gemini 2 允许并行调用，但不提供稳定的 id；Gemini 3 加入了 UUID，使乱序的响应能够干净地关联。

### 流式

三家都支持流式工具调用。线缆格式各异：

- **OpenAI。** `tool_calls[i].function.arguments` 的增量分块逐步到达。你不断累积直到 `finish_reason: "tool_calls"`。
- **Anthropic。** block-start / block-delta / block-stop 事件。`input_json_delta` 分块携带部分参数。
- **Gemini。** `streamFunctionCallArguments`（Gemini 3 中新增）发出带 `functionCallId` 的分块，使多个并行调用可以交错。

Phase 13 · 03 深入讲并行 + 流式重组。本课聚焦声明和单次调用的形态。

### 错误与修复

无效参数错误看起来也不同。

- **OpenAI（非严格）。** 模型返回 `arguments: "{bad json}"`，你的 JSON 解析失败，你注入一条错误消息并重新调用。
- **OpenAI（严格）。** 校验在解码过程中发生；无效 JSON 不可能出现，但可能出现 `refusal`。
- **Anthropic。** `input` 可能包含意外字段；模式是建议性的。在服务端校验。
- **Gemini。** OpenAPI 3.0 的怪癖：对象字段上的 `enum` 被静默忽略；自己校验。

### 翻译器模式

你代码中一份规范的工具声明长这样（形态由你来定）：

```python
Tool(
    name="get_weather",
    description="Use when ...",
    input_schema={"type": "object", "properties": {...}, "required": [...]},
    strict=True,
)
```

三个极小的函数把它翻译成三种厂商形态。`code/main.py` 中的测试框架正是这么做的，然后把一个假的工具调用通过各家厂商的响应形态往返一遍。无需网络 —— 本课教的是形态，而非 HTTP。

生产团队把这个翻译器包装在 `AbstractToolset`（Pydantic AI）、`UniversalToolNode`（LangGraph）或 `BaseTool`（LlamaIndex）中。Phase 13 · 17 交付一个网关，在三家中任意一家之前暴露一个 OpenAI 形态的 API。

## 动手用

`code/main.py` 定义了一个规范的 `Tool` dataclass 和三个翻译器，分别发出 OpenAI、Anthropic 和 Gemini 的声明 JSON。然后它把每种形态的手工构造厂商响应解析成同一个规范调用对象，演示出皮囊之下语义是相同的。运行它，并排对比三份声明。

要看什么：

- 三个声明块只在外壳和字段名上不同。
- 三个响应块在调用所在位置上不同（顶层 `tool_calls`、`content[]` 块、`parts[]` 条目）。
- 一个 `canonical_call()` 函数从全部三种响应形态中提取 `{id, name, args}`。

## 交付

本课产出 `outputs/skill-provider-portability-audit.md`。给定一个针对某一家厂商的函数调用集成，该技能产出一份可移植性审计：它依赖了哪些厂商上限、哪些字段需要重命名，以及移植到其他每家厂商时会有什么崩坏。

## 练习

1. 运行 `code/main.py`，验证三份厂商声明 JSON 都序列化自同一个底层 `Tool` 对象。修改规范工具，添加一个 enum 参数，确认只有 Gemini 翻译器需要处理 OpenAPI 怪癖。

2. 为每家厂商添加一个 `ListToolsResponse` 解析器，提取模型在 `list_tools` 或发现调用之后返回的工具列表。OpenAI 原生没有这个；记录这一不对称性。

3. 实现 `tool_choice` 转换：把一个规范的 `ToolChoice(mode="force", tool_name="x")` 映射成全部三种厂商形态。然后映射 `mode="any"` 和 `mode="none"`。对照本课的差异表。

4. 选择三家厂商之一，从头到尾读它的函数调用指南。在它的模式规范中找出一个另外两家不支持的字段。候选：OpenAI 的 `strict`、Anthropic 的 `disable_parallel_tool_use`、Gemini 的 `function_calling_config.allowed_function_names`。

5. 写一个测试向量：一个其参数违反所声明模式的工具调用。把它通过每家厂商的校验器运行（用 Lesson 01 中的 stdlib 校验器作为代理即可），并记录哪些错误触发。记录你会在生产中为追求严格性而使用哪家厂商。

## 关键术语

| 术语 | 人们怎么说 | 它实际是什么 |
|------|----------------|------------------------|
| Function calling | “工具使用” | 厂商级别用于结构化工具调用发出的 API |
| Tool declaration | “工具规格” | 名称 + 描述 + JSON Schema 输入载荷 |
| `tool_choice` | “强制 / 禁止” | Auto / required / none / 特定名称模式 |
| Strict mode | “模式强制” | 约束解码以匹配模式的 OpenAI 标志 |
| `tool_use` 块 | “Anthropic 的调用形态” | 带 id、name、input 的内联内容块 |
| `functionCall` part | “Gemini 的调用形态” | 包含 name、args 和 id 的一个 `parts[]` 条目 |
| Arguments-as-string | “字符串化的 JSON” | OpenAI 把参数作为 JSON 字符串返回，而非对象 |
| Parallel tool calls | “一轮内扇出” | 一条 assistant 消息中的多个工具调用 |
| Refusal | “模型拒绝” | 严格模式专有的拒绝块，而非一次调用 |
| OpenAPI 3.0 subset | “Gemini 模式怪癖” | Gemini 使用一种类 JSON-Schema 的方言，有细微差异 |

## 延伸阅读

- [OpenAI — Function calling guide](https://platform.openai.com/docs/guides/function-calling) —— 规范参考，包括严格模式和并行调用
- [Anthropic — Tool use overview](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview) —— `tool_use` 和 `tool_result` 块的语义
- [Google — Gemini function calling](https://ai.google.dev/gemini-api/docs/function-calling) —— 并行调用、唯一 id 和 OpenAPI 子集
- [Vertex AI — Function calling reference](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/multimodal/function-calling) —— Gemini 的企业级界面
- [OpenAI — Structured outputs](https://platform.openai.com/docs/guides/structured-outputs) —— 严格模式模式强制的细节
