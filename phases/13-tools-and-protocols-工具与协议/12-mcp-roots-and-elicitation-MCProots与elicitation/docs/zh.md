# Roots 与 Elicitation —— 范围限定与流程中途的用户输入

> 硬编码路径会在用户打开另一个项目的那一刻失效。预填的工具参数会在用户描述不充分时失效。Roots 将服务器限定在一组由用户掌控的 URI 范围内；elicitation 则在工具调用进行到一半时暂停，通过表单或 URL 向用户请求结构化输入。两个客户端原语，针对两种常见的 MCP 失败模式。SEP-1036（URL 模式 elicitation，2025-11-25）在 2026 上半年仍属实验性 —— 依赖它之前先确认 SDK 版本。

**类型：** 构建
**语言：** Python（stdlib，roots + elicitation 演示）
**前置要求：** Phase 13 · 07（MCP 服务器）
**时长：** 约 45 分钟

## 学习目标

- 声明 `roots` 并响应 `notifications/roots/list_changed`。
- 将服务器的文件操作限制在已声明的 root 集合内的 URI 上。
- 使用 `elicitation/create` 在工具调用中途向用户请求确认或结构化输入。
- 在表单模式与 URL 模式 elicitation 之间做出选择（后者属实验性；已标注漂移风险）。

## 问题

一个 notes MCP 服务器在生产环境中会遇到的两个具体失败。

**错误的路径假设。** 服务器是针对 `~/notes` 编写的。一位用户在另一台机器上把笔记放在 `~/Documents/Notes`，于是工具调用会静默失败（找不到文件），或者更糟，写到了错误的位置。

**用户知道但缺失的参数。** 用户说“删掉那条旧的 TPS report 笔记”。模型调用了 `notes_delete(title: "TPS report")`，但有三条匹配的笔记，分别来自 2023、2024 和 2025 年。工具无法猜测。返回“有歧义”很烦人；对这三条全部执行则是灾难性的。

Roots 解决第一个问题：客户端在 `initialize` 阶段声明服务器可触及的 URI 集合。Elicitation 解决第二个问题：服务器暂停工具调用并发送 `elicitation/create`，请用户挑选其中一条。

## 概念

### Roots

客户端在 `initialize` 阶段声明 root 列表：

```json
{
  "capabilities": {"roots": {"listChanged": true}}
}
```

随后服务器可以调用 `roots/list`：

```json
{"roots": [{"uri": "file:///Users/alice/Documents/Notes", "name": "Notes"}]}
```

服务器必须将 roots 视为边界：任何对 root 集合之外的文件读写都应被拒绝。这并非由客户端强制（服务器终究是用户信任的代码），但符合规范的服务器会遵守它。

当用户添加或移除某个 root 时，客户端会发送 `notifications/roots/list_changed`。服务器重新调用 `roots/list` 并更新其边界。

### 为什么 roots 是客户端原语

Roots 由客户端声明，因为它们代表用户的同意模型。是用户告诉 Claude Desktop“给这个 notes 服务器访问这两个目录的权限”。服务器无法扩大这个范围。

### Elicitation：默认的表单模式

`elicitation/create` 接收一个表单模式外加一段自然语言提示：

```json
{
  "method": "elicitation/create",
  "params": {
    "message": "Delete 'TPS report'? Multiple notes match; pick one.",
    "requestedSchema": {
      "type": "object",
      "properties": {
        "note_id": {
          "type": "string",
          "enum": ["note-3", "note-7", "note-14"]
        },
        "confirm": {"type": "boolean"}
      },
      "required": ["note_id", "confirm"]
    }
  }
}
```

客户端渲染一个表单，收集用户的回答，返回：

```json
{
  "action": "accept",
  "content": {"note_id": "note-14", "confirm": true}
}
```

三种可能的 action：`accept`（用户已填写）、`decline`（用户关闭了它）、`cancel`（用户中止了整个工具调用）。

表单模式是扁平的 —— v1 不支持嵌套对象。SDK 通常会拒绝任何比单层更复杂的结构。

### Elicitation：URL 模式（SEP-1036，实验性）

2025-11-25 新增。服务器发送的不是模式，而是一个 URL：

```json
{
  "method": "elicitation/create",
  "params": {
    "message": "Sign in to GitHub",
    "url": "https://github.com/login/oauth/authorize?client_id=..."
  }
}
```

客户端在浏览器中打开该 URL，等待完成，待用户返回后再返回结果。适用于 OAuth 流程、支付授权和文档签署等表单无法胜任的场景。

漂移风险提示：SEP-1036 的响应结构仍在调整中；有些 SDK 返回回调 URL，另一些返回完成令牌。在生产中使用 URL 模式前，请阅读你所用 SDK 的发布说明。

### 何时该用 elicitation

- 在破坏性操作前请求用户确认（破坏性提示 + elicitation）。
- 消歧（从 N 个匹配项中挑一个）。
- 首次运行设置（API key、目录、偏好）。
- OAuth 式流程（URL 模式）。

### 何时不该用 elicitation

- 填充模型本可以用文字询问的工具必填参数。用一次普通的重新提问，而不是 elicitation 对话框。
- 高频调用。Elicitation 会打断对话；不要在循环里触发它。
- 任何服务器可以事后校验的内容。先校验，返回错误，让模型用文字向用户询问。

### 人在回路（Human-in-the-loop）的桥梁

Elicitation 与 sampling 合在一起，构成了 MCP 的“人在回路”模型。服务器的智能体循环可以为用户输入（elicitation）或模型推理（sampling）而暂停。Phase 13 · 11 讲了 sampling；本课讲 elicitation。把它们结合起来即可获得完整的循环中途控制能力。

## 上手实践

`code/main.py` 为 notes 服务器扩展了以下内容：

- 一个 `roots/list` 响应，服务器在收到 root-list-changed 通知后会重新查询它。
- 一个 `notes_delete` 工具，在多条笔记匹配时使用 `elicitation/create` 进行消歧。
- 一个 `notes_setup` 工具，使用 URL 模式 elicitation 打开首次运行的配置页面（模拟）。
- 一个边界检查，拒绝对已声明 roots 之外的 URI 执行操作。

该演示运行三个场景：顺利路径（一条匹配）、消歧（三条匹配，触发 elicitation）、越界写入（被拒绝）。

## 交付物

本课产出 `outputs/skill-elicitation-form-designer.md`。给定一个可能需要用户确认或消歧的工具，该 skill 会设计 elicitation 表单模式以及消息模板。

## 练习

1. 运行 `code/main.py`。触发消歧路径；确认模拟用户的回答会被路由回工具。

2. 添加一个新工具 `notes_archive`，每次都要求 elicitation 确认（破坏性提示）。检查 UX：与模型用文字重新询问相比，体验如何？

3. 为首次运行的 OAuth 流程实现 URL 模式 elicitation。注意漂移风险，并加上一个 SDK 版本守卫。

4. 扩展 `roots/list` 处理逻辑：当通知到达时，服务器应原子地重新读取并重新扫描那些此刻可能已越界的打开文件句柄。

5. 阅读 GitHub 上 SEP-1036 的议题讨论帖。找出一个会影响服务器如何处理 URL 模式回调的悬而未决的问题。

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| Root | “同意边界” | 客户端允许服务器触及的 URI |
| `roots/list` | “服务器请求范围” | 客户端返回当前的 root 集合 |
| `notifications/roots/list_changed` | “用户更改了范围” | 客户端发出信号，表示 root 集合已变更 |
| Elicitation | “在调用中途询问用户” | 服务器发起的、请求结构化用户输入的请求 |
| `elicitation/create` | “那个方法” | 用于 elicitation 请求的 JSON-RPC 方法 |
| 表单模式 | “模式驱动的表单” | 在客户端 UI 中渲染为表单的扁平 JSON Schema |
| URL 模式 | “浏览器重定向” | SEP-1036 实验性；打开一个 URL 并等待 |
| `accept` / `decline` / `cancel` | “用户响应结果” | 服务器需处理的三个分支 |
| 消歧 | “挑一个” | 工具有 N 个候选项时常见的 elicitation 用例 |
| 扁平表单 | “仅顶层属性” | elicitation 模式不能嵌套 |

## 延伸阅读

- [MCP — Client roots spec](https://modelcontextprotocol.io/specification/draft/client/roots) —— roots 的权威参考
- [MCP — Client elicitation spec](https://modelcontextprotocol.io/specification/draft/client/elicitation) —— elicitation 的权威参考
- [Cisco — What's new in MCP elicitation, structured content, OAuth enhancements](https://blogs.cisco.com/developer/whats-new-in-mcp-elicitation-structured-content-and-oauth-enhancements) —— 2025-11-25 新增内容的逐项讲解
- [MCP — GitHub SEP-1036](https://github.com/modelcontextprotocol/modelcontextprotocol) —— URL 模式 elicitation 提案（实验性，有漂移风险）
- [The New Stack — How elicitation brings human-in-the-loop to AI tools](https://thenewstack.io/how-elicitation-in-mcp-brings-human-in-the-loop-to-ai-tools/) —— UX 演练
