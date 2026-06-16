# MCP 基础 —— 基本单元、生命周期、JSON-RPC 基础

> 在 MCP 之前，每一次集成都是一次性的。Model Context Protocol（模型上下文协议）由 Anthropic 于 2024 年 11 月首次发布，如今由 Linux 基金会旗下的 Agentic AI Foundation 管理，它标准化了发现与调用，使任何客户端都能与任何服务器通信。2025-11-25 规范定义了六个基本单元（三个服务器端，三个客户端）、一个三阶段生命周期，以及一种 JSON-RPC 2.0 线路格式。学会这些，本阶段 MCP 章节的其余部分就只剩阅读了。

**类型：** 学习
**语言：** Python（标准库，JSON-RPC 解析器）
**前置知识：** Phase 13 · 01 至 05（工具接口与函数调用）
**时长：** 约 45 分钟

## 学习目标

- 说出全部六个 MCP 基本单元（服务器端的 tools、resources、prompts；客户端的 roots、sampling、elicitation），并各举一个使用场景。
- 走通三阶段生命周期（initialize、operation、shutdown），并说明每个阶段由谁发送哪条消息。
- 解析并生成 JSON-RPC 2.0 的请求、响应和通知信封。
- 解释 `initialize` 时的能力协商是什么，以及没有它会出什么问题。

## 问题所在

在 MCP 之前，每个使用工具的智能体都有自己的协议。Cursor 有一个形似 MCP 但不兼容的工具系统。Claude Desktop 自带的又是另一套。VS Code 的 Copilot 扩展又是第三套。一个构建了"Postgres 查询"工具的团队，把同一个工具写了三遍，每一遍都针对不同宿主的 API。要复用它就得复制代码。

结果是一次性集成的寒武纪大爆发，以及生态系统发展速度的天花板。

MCP 通过标准化线路格式解决了这个问题。单个 MCP 服务器可以在每个 MCP 客户端中工作：Claude Desktop、ChatGPT、Cursor、VS Code、Gemini、Goose、Zed、Windsurf，到 2026 年 4 月已有 300 多个客户端。SDK 每月下载量 1.1 亿次。1 万多个公开服务器。Linux 基金会于 2025 年 12 月在新成立的 Agentic AI Foundation 下接管了管理工作。

本阶段所用的规范修订版本是 **2025-11-25**。它新增了异步 Tasks（SEP-1686）、URL 模式 elicitation（SEP-1036）、带工具的 sampling（SEP-1577）、增量作用域同意（SEP-835），以及 OAuth 2.1 资源指示符语义。Phase 13 · 09 至 16 涵盖这些扩展。本课止步于基础部分。

## 概念

### 三个服务器端基本单元

1. **Tools（工具）。** 可调用的动作。与 Phase 13 · 01 中相同的四步循环。
2. **Resources（资源）。** 暴露的数据。只读内容，可通过 URI 寻址：`file:///path`、`db://query/...`、自定义方案。
3. **Prompts（提示）。** 可复用模板。宿主 UI 中的斜杠命令；服务器提供模板，客户端填入参数。

### 三个客户端基本单元

4. **Roots（根）。** 允许服务器触及的 URI 集合。客户端声明它们；服务器遵守它们。
5. **Sampling（采样）。** 服务器请求客户端的模型执行一次补全。使得服务器托管的智能体循环无需服务器端 API 密钥即可运行。
6. **Elicitation（征询）。** 服务器在流程中途向客户端的用户索取结构化输入。表单或 URL（SEP-1036）。

MCP 中的每一个能力都恰好属于这六者之一。Phase 13 · 10 至 14 深入讲解每一个。

### 线路格式：JSON-RPC 2.0

每条消息都是一个带有以下字段的 JSON 对象：

- 请求：`{jsonrpc: "2.0", id, method, params}`。
- 响应：`{jsonrpc: "2.0", id, result | error}`。
- 通知：`{jsonrpc: "2.0", method, params}` —— 无 `id`，不期待响应。

基础规范有约 15 个方法，按基本单元分组。重要的有：

- `initialize` / `initialized`（握手）
- `tools/list`、`tools/call`
- `resources/list`、`resources/read`、`resources/subscribe`
- `prompts/list`、`prompts/get`
- `sampling/createMessage`（服务器到客户端）
- `notifications/tools/list_changed`、`notifications/resources/updated`、`notifications/progress`

### 三阶段生命周期

**阶段 1：initialize。**

客户端发送 `initialize`，附带其 `capabilities` 和 `clientInfo`。服务器以自己的 `capabilities`、`serverInfo` 以及它所讲的规范版本作出响应。客户端在消化完响应后发送 `notifications/initialized`。从此以后，任何一方都可以按协商好的能力发送请求。

**阶段 2：operation。**

双向。客户端调用 `tools/list` 进行发现，然后调用 `tools/call` 进行调用。如果服务器声明了该能力，它可以发送 `sampling/createMessage`。当服务器的工具集发生变化时，它可以发送 `notifications/tools/list_changed`。当用户更改根作用域时，客户端可以发送 `notifications/roots/list_changed`。

**阶段 3：shutdown。**

任何一方关闭传输。MCP 中没有结构化的关闭方法；由传输层（stdio 或 Streamable HTTP，Phase 13 · 09）承载连接结束信号。

### 能力协商

`initialize` 握手中的 `capabilities` 就是契约。来自服务器的示例：

```json
{
  "tools": {"listChanged": true},
  "resources": {"subscribe": true, "listChanged": true},
  "prompts": {"listChanged": true}
}
```

服务器声明它可以发出 `tools/list_changed` 通知并支持 `resources/subscribe`。客户端通过声明自己的能力来表示同意：

```json
{
  "roots": {"listChanged": true},
  "sampling": {},
  "elicitation": {}
}
```

如果客户端不声明 `sampling`，服务器就不得调用 `sampling/createMessage`。对称地：如果服务器不声明 `resources.subscribe`，客户端就不得尝试订阅。

这正是防止生态系统漂移的机制。不支持 sampling 的客户端仍然是合法的 MCP 客户端；不调用 `sampling` 的服务器仍然是合法的 MCP 服务器。它们只是不一起使用那个特性而已。

### 结构化内容与错误形态

`tools/call` 返回一个由类型化块组成的 `content` 数组：`text`、`image`、`resource`。Phase 13 · 14 向该列表新增 MCP Apps（`ui://` 交互式 UI）。

错误使用 JSON-RPC 错误码。规范定义的新增项：`-32002` "Resource not found"、`-32603` "Internal error"，以及作为 `error.data` 的 MCP 特定错误数据。

### 客户端能力 vs 工具调用细节

一个常见的混淆：`capabilities.tools` 表示客户端是否支持工具列表变更通知。客户端是否会调用特定工具，是由其模型驱动的运行时选择，而非能力标志。能力标志是规范层面的契约。模型的选择与之正交。

### 为什么选 JSON-RPC 而不是 REST？

JSON-RPC 2.0（2010 年）是一种轻量的双向协议。REST 由客户端发起。MCP 需要由服务器发起的消息（sampling、通知），所以具备对称请求/响应形态的 JSON-RPC 是自然的选择。JSON-RPC 还能在 stdio 和 WebSocket/Streamable HTTP 之上干净地组合，无需重新发明 HTTP 的请求形态。

```figure
mcp-tool-call
```

## 动手用

`code/main.py` 提供了一个最小化的 JSON-RPC 2.0 解析器和生成器，然后手动走一遍 `initialize` → `tools/list` → `tools/call` → `shutdown` 序列，打印每一条消息。没有真实的传输；只有消息形态。对照 Further Reading 中链接的规范，验证每个信封。

要关注的地方：

- `initialize` 双向声明能力；响应中有 `serverInfo` 和 `protocolVersion: "2025-11-25"`。
- `tools/list` 返回一个 `tools` 数组；每个条目都有 `name`、`description`、`inputSchema`。
- `tools/call` 使用 `params.name` 和 `params.arguments`。
- 响应的 `content` 是一个由 `{type, text}` 块组成的数组。

## 交付物

本课产出 `outputs/skill-mcp-handshake-tracer.md`。给定一段 pcap 风格的 MCP 客户端-服务器交互记录，该技能为每条消息标注它属于哪个基本单元、哪个生命周期阶段，以及它依赖哪个能力。

## 练习

1. 运行 `code/main.py`。找出发生能力协商的那一行，并描述如果服务器没有声明 `tools.listChanged` 会有什么变化。

2. 扩展解析器以处理 `notifications/progress`。消息形态：`{method: "notifications/progress", params: {progressToken, progress, total}}`。在一个长时间运行的 `tools/call` 进行中时发出它，并确认客户端处理程序会显示进度条。

3. 从头到尾通读 MCP 2025-11-25 规范 —— 整篇文档约 80 页。找出大多数服务器都不需要的那一个能力标志。提示：它与资源订阅有关。

4. 在纸上勾画一个假想的"定时任务"特性会属于哪个基本单元。（提示：服务器希望客户端在某个预定时间调用它。今天这六个基本单元都不合适。）MCP 的 2026 年路线图为此有一份 SEP 草案。

5. 解析 GitHub 上某个开放 MCP 服务器的一份会话日志。统计请求、响应、通知消息各有多少条。计算流量中生命周期 vs operation 各占多少比例。

## 关键术语

| 术语 | 人们怎么说 | 它实际是什么 |
|------|----------------|------------------------|
| MCP | "Model Context Protocol" | 用于模型到工具发现与调用的开放协议 |
| 服务器端基本单元 | "服务器暴露什么" | tools（动作）、resources（数据）、prompts（模板） |
| 客户端基本单元 | "客户端让服务器使用什么" | roots（作用域）、sampling（LLM 回调）、elicitation（用户输入） |
| JSON-RPC 2.0 | "线路格式" | 对称的请求/响应/通知信封 |
| `initialize` 握手 | "能力协商" | 第一对消息；服务器和客户端声明各自支持的特性 |
| `tools/list` | "发现" | 客户端向服务器索取其当前工具集 |
| `tools/call` | "调用" | 客户端请求服务器带参数执行一个工具 |
| `notifications/*_changed` | "变更事件" | 服务器告知客户端其基本单元列表已变化 |
| 内容块 | "类型化结果" | 工具结果中的 `{type: "text" \| "image" \| "resource" \| "ui_resource"}` |
| SEP | "Spec Evolution Proposal" | 具名的草案提议（例如用于异步 Tasks 的 SEP-1686） |

## 延伸阅读

- [Model Context Protocol — Specification 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25) —— 权威规范文档
- [Model Context Protocol — Architecture concepts](https://modelcontextprotocol.io/docs/concepts/architecture) —— 六基本单元的思维模型
- [Anthropic — Introducing the Model Context Protocol](https://www.anthropic.com/news/model-context-protocol) —— 2024 年 11 月发布博文
- [MCP blog — First MCP anniversary](https://blog.modelcontextprotocol.io/posts/2025-11-25-first-mcp-anniversary/) —— 一周年回顾与 2025-11-25 规范变更
- [WorkOS — MCP 2025-11-25 spec update](https://workos.com/blog/mcp-2025-11-25-spec-update) —— SEP-1686、1036、1577、835 和 1724 的摘要
