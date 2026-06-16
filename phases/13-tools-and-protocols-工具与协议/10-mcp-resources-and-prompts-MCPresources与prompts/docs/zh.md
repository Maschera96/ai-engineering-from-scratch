# MCP Resources 与 Prompts —— 超越工具的上下文暴露

> 工具占据了 MCP 关注度的 90%。另外两个服务端原语解决的是不同的问题。Resources 暴露数据供读取；prompts 把可复用模板暴露为斜杠命令（slash-command）。很多服务端应该用 resources 而非把读取操作包装成工具，应该用 prompts 而非把工作流硬编码进客户端提示词。本课给出决策规则，并逐一讲解 `resources/*` 与 `prompts/*` 消息。

**类型：** Build
**语言：** Python（stdlib，resource + prompt handler）
**前置条件：** Phase 13 · 07（MCP 服务端）
**时间：** ~45 分钟

## 学习目标

- 针对给定领域，在把某能力暴露为工具、resource 还是 prompt 之间做出抉择。
- 实现 `resources/list`、`resources/read`、`resources/subscribe`，并处理 `notifications/resources/updated`。
- 实现带参数模板的 `prompts/list` 与 `prompts/get`。
- 识别宿主（host）何时将 prompts 呈现为斜杠命令，何时自动注入为上下文。

## 问题

一个朴素的笔记应用 MCP 服务端把一切都暴露为工具：`notes_read`、`notes_list`、`notes_search`。这把每一次数据访问都包装成由模型驱动的工具调用。后果是：

- 对每个可能受益于上下文的查询，模型都得自己决定是否调用 `notes_read`。
- 只读内容无法被订阅，也无法流式推送到宿主的侧边栏。
- 客户端 UI（Claude Desktop 的资源附加面板、Cursor 的 “Include file” 选择器）无法呈现这些数据。

正确的划分是：把数据暴露为 resource，把可变或需计算的动作暴露为工具，把可复用的多步骤工作流暴露为 prompt。每个原语都有其 UX 表现形式和访问模式。

## 概念

### 工具 vs resources vs prompts —— 决策规则

| 能力 | 原语 |
|------------|-----------|
| 用户想搜索、过滤或变换数据 | tool |
| 用户想让宿主把这份数据作为上下文纳入 | resource |
| 用户想要一个可复跑的模板化工作流 | prompt |

经验法则：如果模型在每个相关查询里调用它都有好处，那它就是工具。如果用户把它附加到对话里有好处，那它就是 resource。如果用户想复用的单元是一整套多步骤工作流，那它就是 prompt。

### Resources

`resources/list` 返回 `{resources: [{uri, name, mimeType, description?}]}`。`resources/read` 接收 `{uri}` 并返回 `{contents: [{uri, mimeType, text | blob}]}`。

URI 可以是任何可寻址的东西：

- `file:///Users/alice/notes/mcp.md`
- `postgres://my-db/query/SELECT ...`
- `notes://note-14`（自定义 scheme）
- `memory://session-2026-04-22/recent`（服务端特定）

`contents[]` 同时支持文本和二进制。二进制使用 `blob` 作为 base64 编码字符串，外加一个 `mimeType`。

### 资源订阅

在能力中声明 `{resources: {subscribe: true}}`。客户端调用 `resources/subscribe {uri}`。资源变化时服务端发送 `notifications/resources/updated {uri}`。客户端重新读取。

用例：一个笔记服务端，其资源是磁盘上的文件；文件监视器触发更新通知；当文件在宿主之外被编辑时，Claude Desktop 重新把该文件拉入上下文。

### 资源模板（2025-11-25 新增）

`resourceTemplates` 让你能暴露一个参数化的 URI 模式：`notes://{id}`，其中 `id` 是补全目标。客户端可以在资源选择器中对 id 进行自动补全。

### Prompts

`prompts/list` 返回 `{prompts: [{name, description, arguments?}]}`。`prompts/get` 接收 `{name, arguments}` 并返回 `{description, messages: [{role, content}]}`。

一个 prompt 是一个模板，它会填充成一组消息，由宿主喂给其模型。例如，一个 `code_review` prompt 接收一个 `file_path` 参数，返回一个三条消息的序列：一条系统消息、一条带文件正文的用户消息，以及一条带推理模板的助手开场白。

### 宿主与 prompts

Claude Desktop、VS Code 和 Cursor 把 prompts 呈现为聊天 UI 中的斜杠命令。用户输入 `/code_review`，然后从表单中挑选参数。服务端的 prompt 就是 “用户快捷方式” 与 “发给模型的完整提示词” 之间的契约。

并非每个客户端都已支持 prompts —— 检查能力协商（capability negotiation）。一个声明了 prompt 能力的服务端，若搭配一个不支持 prompt 的客户端，就只是看不到这些斜杠命令而已。

### “list changed” 通知

当集合发生变化时，resources 和 prompts 都会发出 `notifications/list_changed`。一个刚导入了 20 条新笔记的笔记服务端会发出 `notifications/resources/list_changed`；客户端重新调用 `resources/list` 来获取新增项。

### 内容类型约定

文本：`mimeType: "text/plain"`、`text/markdown`、`application/json`。
二进制：`image/png`、`application/pdf`，外加 `blob` 字段。
MCP Apps（第 14 课）：在 `ui://` URI 中使用 `text/html;profile=mcp-app`。

### 动态资源

资源 URI 不必对应一个静态文件。`notes://recent` 可以在每次读取时返回最新的五条笔记。`db://query/users/active` 可以执行一个参数化查询。服务端可以自由地动态计算内容。

规则：如果客户端可以按 URI 缓存，那么 URI 必须是稳定的。如果计算是一次性的，那么 URI 应包含时间戳或 nonce，以免客户端缓存失效后取到陈旧数据。

### 订阅 vs 轮询

支持订阅的客户端通过 `notifications/resources/updated` 获得服务端推送。不支持订阅的客户端或宿主则通过重新读取来轮询。两者都符合规范。服务端的能力声明告诉客户端它支持哪一种。

订阅的代价：服务端上的每会话状态（谁订阅了什么）。保持订阅集合有界；断开连接的客户端应当超时。

### Prompts vs 系统提示词

MCP 中的 prompts 不是系统提示词。宿主的系统提示词（它自己的运行指令）和 MCP prompts（由用户调用的服务端提供的模板）并列存在。一个行为良好的客户端绝不会让服务端 prompt 覆盖它自己的系统提示词；它会把两者分层叠加。

## 动手用

`code/main.py` 在第 07 课的笔记服务端基础上扩展了：

- 每条笔记对应的资源（`notes://note-1` 等），支持 `resources/subscribe`。
- 一个 `review_note` prompt，渲染成一个三条消息的模板。
- 一个文件监视器模拟，在笔记被修改时发出 `notifications/resources/updated`。
- 一个 `notes://recent` 动态资源，总是返回最新的五条笔记。

运行 demo 以查看完整流程。

## 交付

本课产出 `outputs/skill-primitive-splitter.md`。给定一个提议的 MCP 服务端，该 skill 将每个能力归类为 tool / resource / prompt，并给出理由。

## 练习

1. 运行 `code/main.py`。观察初始资源列表，然后触发一次笔记编辑，验证 `notifications/resources/updated` 事件被触发。

2. 添加一个 `resources/list_changed` 发射器：当新建一条笔记时，发送该通知以便客户端重新发现。

3. 为一个 GitHub MCP 服务端设计三个 prompts：`summarize_pr`、`triage_issue`、`release_notes`。每个都带参数模式（argument schema）。prompt 主体应当无需进一步编辑即可运行。

4. 取第 07 课服务端中现有的一个工具，判定它应保持为工具，还是应拆分为一对 resource 加 tool。用一句话论证。

5. 阅读规范的 `server/resources` 和 `server/prompts` 章节。找出 `resources/read` 中那个很少被填充但规范支持的字段。提示：看看资源内容上的 `_meta`。

## 关键术语

| 术语 | 人们怎么说 | 它实际指什么 |
|------|----------------|------------------------|
| Resource | “暴露的数据” | 宿主可读取的、可按 URI 寻址的内容 |
| 资源 URI | “指向数据的指针” | 带 scheme 前缀的标识符（`file://`、`notes://` 等） |
| `resources/subscribe` | “监听变化” | 客户端主动开启的、针对特定 URI 的服务端推送更新 |
| `notifications/resources/updated` | “资源变了” | 向客户端发出的信号，表示某个已订阅资源有了新内容 |
| 资源模板 | “参数化 URI” | 带有补全提示、供宿主选择器使用的 URI 模式 |
| Prompt | “斜杠命令模板” | 带参数槽的具名多消息模板 |
| Prompt 参数 | “模板输入” | 宿主在渲染前收集的有类型参数 |
| `prompts/get` | “渲染模板” | 服务端返回填充好的消息列表 |
| Content block | “有类型的块” | `{type: text \| image \| resource \| ui_resource}` |
| 斜杠命令 UX | “用户快捷方式” | 宿主把 prompts 呈现为以 `/` 开头的命令 |

## 延伸阅读

- [MCP — Concepts: Resources](https://modelcontextprotocol.io/docs/concepts/resources) —— 资源 URI、订阅与模板
- [MCP — Concepts: Prompts](https://modelcontextprotocol.io/docs/concepts/prompts) —— prompt 模板与斜杠命令集成
- [MCP — Server resources spec 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25/server/resources) —— 完整的 `resources/*` 消息参考
- [MCP — Server prompts spec 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25/server/prompts) —— 完整的 `prompts/*` 消息参考
- [MCP — Protocol info site: resources](https://modelcontextprotocol.info/docs/concepts/resources/) —— 对官方文档加以扩展的社区指南
