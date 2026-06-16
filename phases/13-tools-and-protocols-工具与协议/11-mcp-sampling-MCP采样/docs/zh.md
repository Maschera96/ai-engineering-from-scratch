# MCP 采样——服务端发起的 LLM 补全与智能体循环

> 大多数 MCP 服务器都是没有判断力的执行器：接收参数、运行代码、返回内容。采样让服务器可以反转方向：它请求客户端的 LLM 做出决策。这使得服务端托管的智能体循环成为可能，而服务器无需持有任何模型凭据。SEP-1577 于 2025-11-25 合并，在采样请求中加入了工具，使该循环能够包含更深入的推理。漂移风险提示：SEP-1577 的「采样中嵌入工具」形态在整个 2026 年第一季度都还处于实验阶段，且在各 SDK API 中仍在逐步定型。

**类型：** 构建
**语言：** Python（标准库，采样脚手架）
**前置要求：** Phase 13 · 07（MCP 服务器）、Phase 13 · 10（资源与提示）
**时长：** ~75 分钟

## 学习目标

- 解释 `sampling/createMessage` 解决了什么问题（服务端托管循环，无需服务端 API 密钥）。
- 实现一个服务器，让它请求客户端对一段多轮提示进行采样并返回补全结果。
- 使用 `modelPreferences`（成本 / 速度 / 智能优先级）来引导客户端的模型选择。
- 构建一个 `summarize_repo` 工具，让它通过采样在内部迭代，而不是把行为写死。

## 问题

一个用于代码摘要工作流的实用 MCP 服务器需要做到：遍历文件树、挑选要读取的文件、综合出摘要并返回。LLM 推理在哪里发生？

方案 A：服务器调用它自己的 LLM。需要 API 密钥，在服务端计费，每个用户成本都很高。

方案 B：服务器返回原始内容；由客户端的智能体来做推理。可行，但把服务器逻辑搬进了客户端提示中，这很脆弱。

方案 C：服务器通过 `sampling/createMessage` 请求客户端的 LLM。服务器保留算法（读哪些文件、做几遍），而客户端保留计费和模型选择权。服务器完全没有任何凭据。

采样就是方案 C。它是一种机制，让受信任的服务器能够托管智能体循环，而无需自己成为一个完整的 LLM 宿主。

## 概念

### `sampling/createMessage` 请求

服务器发送：

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "method": "sampling/createMessage",
  "params": {
    "messages": [{"role": "user", "content": {"type": "text", "text": "..."}}],
    "systemPrompt": "...",
    "includeContext": "none",
    "modelPreferences": {
      "costPriority": 0.3,
      "speedPriority": 0.2,
      "intelligencePriority": 0.5,
      "hints": [{"name": "claude-3-5-sonnet"}]
    },
    "maxTokens": 1024
  }
}
```

客户端运行其 LLM，返回：

```json
{"jsonrpc": "2.0", "id": 42, "result": {
  "role": "assistant",
  "content": {"type": "text", "text": "..."},
  "model": "claude-3-5-sonnet-20251022",
  "stopReason": "endTurn"
}}
```

### `modelPreferences`

三个相加等于 1.0 的浮点数：

- `costPriority`：倾向更便宜的模型。
- `speedPriority`：倾向更快的模型。
- `intelligencePriority`：倾向更强的模型。

外加 `hints`：服务器偏好的具名模型。客户端可以遵从也可以不遵从这些提示；客户端的用户配置始终优先。

### `includeContext`

三个取值：

- `"none"` —— 仅包含服务器提供的消息。默认值。
- `"thisServer"` —— 包含此服务器会话中先前的消息。
- `"allServers"` —— 包含所有会话上下文。

自 2025-11-25 起，`includeContext` 被软性弃用，因为它会泄露跨服务器上下文，这是一个安全隐患。优先使用 `"none"`，并在消息中显式传入上下文。

### 带工具的采样（SEP-1577）

2025-11-25 新增：采样请求可以包含一个 `tools` 数组。客户端使用这些工具运行完整的工具调用循环。这让服务器能够通过客户端的模型托管一个 ReAct 风格的智能体循环。

```json
{
  "messages": [...],
  "tools": [
    {"name": "fetch_url", "description": "...", "inputSchema": {...}}
  ]
}
```

客户端循环：采样、若被调用则执行工具、再次采样、返回最终的助手消息。这一特性在整个 2026 年第一季度都属于实验性质；SDK 签名可能仍会漂移。实现时请对照 2025-11-25 规范的 client/sampling 章节进行确认。

### 人在回路（Human-in-the-loop）

客户端在运行采样之前，必须向用户展示服务器请求模型去做什么。恶意服务器可能利用采样来操纵用户的会话（「对用户说 X，好让他们点击 Y」）。Claude Desktop、VS Code 和 Cursor 都会以确认对话框的形式呈现采样请求，用户可以拒绝。

2026 年的共识：没有人工确认的采样是一个危险信号。网关（Phase 13 · 17）可以自动批准低风险采样，并自动拒绝任何可疑请求。

### 无需 API 密钥的服务端托管循环

典型用例：一个本身没有任何 LLM 访问权限的代码摘要 MCP 服务器。它会：

1. 遍历仓库结构。
2. 调用 `sampling/createMessage`，提示「挑选五个最可能描述该仓库用途的文件」。
3. 读取这些文件。
4. 调用 `sampling/createMessage`，附上这些文件的内容并提示「用 3 段话总结该仓库」。
5. 把摘要作为 `tools/call` 结果返回。

服务器从不接触任何 LLM API。客户端的用户用自己的凭据为这些补全付费。

### 安全风险（Unit 42 披露，2026 年第一季度）

- **隐蔽采样。** 一个工具总是带着「用会话上下文中用户的电子邮件作答」来调用采样。Phase 13 · 15 涵盖了这些攻击向量。
- **通过采样窃取资源。** 服务器请求客户端去总结攻击者的载荷，费用却由用户买单。
- **循环炸弹。** 服务器在一个紧密循环中调用采样。客户端必须强制执行按会话计的速率限制。

## 使用它

`code/main.py` 提供了一个模拟的服务器到客户端采样脚手架。一个被模拟的 `summarize_repo` 工具会发起两轮采样（先挑文件，再做总结），而模拟客户端返回预设好的响应。该脚手架展示了：

- 服务器带着 `modelPreferences` 发送 `sampling/createMessage`。
- 客户端返回一个补全。
- 服务器继续其循环。
- 速率限制器对每次工具调用的总采样次数设置上限。

需要重点观察的内容：

- 服务器只暴露一个工具（`summarize_repo`）；所有推理都发生在采样调用中。
- 模型偏好会影响客户端的模型选择；hints 列出偏好的模型。
- 循环在 `stopReason: "endTurn"` 时终止。
- `max_samples_per_tool = 5` 这一上限会拦住失控的循环。

## 交付它

本课产出 `outputs/skill-sampling-loop-designer.md`。给定一个需要 LLM 调用的服务端算法（研究、摘要、规划），该技能会设计一个基于采样的实现，配以合适的 modelPreferences、速率限制和安全确认。

## 练习

1. 运行 `code/main.py`。把 `max_samples_per_tool` 改为 2，观察速率限制触发的截断。

2. 实现 SEP-1577 的「采样中嵌入工具」变体：采样请求携带一个 `tools` 数组。验证客户端的循环会在返回最终补全之前执行这些工具。注意漂移风险：SDK 签名在 2026 年上半年可能仍会变化。

3. 加入人在回路确认：在服务器的第一个 `sampling/createMessage` 之前，暂停并等待用户批准。被拒绝的调用返回一个带类型的拒绝结果。

4. 加入一个按客户端会话作为键的按用户速率限制器。同一用户在同一服务器上的循环应共享同一份预算。

5. 设计一个 `summarize_pdf` 工具，用采样来挑选要纳入的分块。勾勒出发送的消息。当 `modelPreferences.intelligencePriority` 取 0.1 与取 0.9 时，行为会如何改变？

## 关键术语

| 术语 | 人们怎么说 | 它实际的含义 |
|------|----------------|------------------------|
| 采样 | 「服务器到客户端的 LLM 调用」 | 服务器请求客户端的模型给出一个补全 |
| `sampling/createMessage` | 「那个方法」 | 用于采样请求的 JSON-RPC 方法 |
| `modelPreferences` | 「模型优先级」 | 成本 / 速度 / 智能权重外加名称提示 |
| `includeContext` | 「跨会话泄露」 | 被软性弃用的上下文纳入模式 |
| SEP-1577 | 「采样中的工具」 | 允许在采样中嵌入工具以实现服务端托管的 ReAct |
| 人在回路 | 「用户确认」 | 客户端在运行前向用户呈现采样请求 |
| 循环炸弹 | 「失控的采样」 | 服务端的无限采样循环；客户端必须做速率限制 |
| 隐蔽采样 | 「隐藏的推理」 | 恶意服务器把意图藏在采样提示里 |
| 资源窃取 | 「占用用户的 LLM 预算」 | 服务器强迫客户端为它并不想要的采样付费 |
| `stopReason` | 「生成为何停止」 | `endTurn`、`stopSequence` 或 `maxTokens` |

## 延伸阅读

- [MCP — Concepts: Sampling](https://modelcontextprotocol.io/docs/concepts/sampling) —— 采样的高层概览
- [MCP — Client sampling spec 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25/client/sampling) —— 规范化的 `sampling/createMessage` 形态
- [MCP — GitHub SEP-1577](https://github.com/modelcontextprotocol/modelcontextprotocol) —— 关于采样中嵌入工具的规范演进提案（实验性）
- [Unit 42 — MCP attack vectors](https://unit42.paloaltonetworks.com/model-context-protocol-attack-vectors/) —— 隐蔽采样与资源窃取模式
- [Speakeasy — MCP sampling core concept](https://www.speakeasy.com/mcp/core-concepts/sampling) —— 附带客户端代码示例的逐步讲解
