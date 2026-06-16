---
name: provider-portability-audit-zh
description: 审计针对某一个提供商的函数调用集成，找出移植到其他两个提供商时会出现的破坏点。
version: 1.0.0
phase: 13
lesson: 02
tags: [function-calling, openai, anthropic, gemini, portability]
---

给定一个基于某个提供商（OpenAI、Anthropic 或 Gemini）的函数调用集成，产出一份可移植性审计报告，列出当同样的逻辑发布到其他两个提供商时出现的每一处字段重命名、行为差异和硬性限制冲突。

产出：

1. 声明差异。对于集成中的每个工具，展示移植到其他两个提供商各自所需的封装 / 字段重命名 / 模式转换。标记出目标提供商不支持的任何 JSON Schema 构造（Gemini：OpenAPI 3.0 子集；OpenAI strict：不支持 `$ref`，不支持有歧义的 `oneOf`）。
2. 响应差异。记录工具调用在每个提供商响应结构中的位置（`tool_calls[]` 还是 `content[]` 块还是 `parts[]` 条目），以及由谁负责解析 `arguments`（OpenAI 上是字符串，Anthropic 和 Gemini 上是对象）。
3. `tool_choice` 差异。将集成当前的选择设置（auto / forbid / force / required）映射到目标提供商的形态；标记出缺失的模式。
4. 限制冲突。报告工具数量（128 / 64 / 64）、模式深度（5 / 10 / 实际上无上限）和单个参数长度上限。对任何超出目标提供商限制的集成，提升为 block 级别严重性。
5. strict 模式映射。说明 strict 模式语义在目标上是否被保留。OpenAI 的 `strict: true` 在 Anthropic 上没有完全对应；Gemini 的 `responseSchema` 是近似实现，但作用在请求层级。

硬性拒绝：
- 任何在非 OpenAI 目标上假设 `arguments` 是字符串的集成。会悄无声息地产生错误结果。
- 任何在移植到 Anthropic 或 Gemini 时工具数量超过 64 且没有路由器的集成。
- 任何在目标为 OpenAI strict 模式时在模式中使用 `$ref` 的集成。

拒绝规则：
- 如果被要求移植一个依赖某提供商特有、且没有对应物的功能的集成（例如 OpenAI Responses API 的有状态轮次、Anthropic 的 computer-use 块），应拒绝并说明哪个功能在目标上没有等价物。
- 如果被要求选出一个赢家，应拒绝。这个选择取决于宿主对 strict 模式的需求、成本画像和并行调用要求。

输出：一页式审计，包含一张按工具列出的差异表、一张限制表，以及针对每个目标提供商的最终“移植裁定”（ship / needs-router / blocked-by-feature）。以一句话结尾，指出杠杆最高的迁移变更。
