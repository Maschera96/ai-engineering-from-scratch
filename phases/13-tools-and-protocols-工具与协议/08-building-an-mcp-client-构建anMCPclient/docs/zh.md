# 构建一个 MCP Client —— 发现、调用与会话管理

> 大多数 MCP 内容都在讲服务器教程，对客户端则一笔带过。客户端代码才是困难编排的所在：进程派生、能力协商、跨多个服务器的工具列表合并、采样回调、重连，以及命名空间冲突解决。本课构建一个多服务器客户端，将三个不同的 MCP 服务器提升到一个扁平的工具命名空间中供模型使用。

**类型：** Build
**语言：** Python（标准库，多服务器 MCP 客户端）
**前置条件：** Phase 13 · 07（构建一个 MCP 服务器）
**时间：** ~75 分钟

## 学习目标

- 将一个 MCP 服务器派生为子进程，完成 `initialize`，并发送一个 `notifications/initialized`。
- 维护每个服务器的会话状态（能力、工具列表、最后看到的通知 id）。
- 将跨多个服务器的工具列表合并到一个命名空间，并处理冲突。
- 将工具调用路由到拥有它的服务器，并重组响应。

## 问题

一个真实的智能体宿主（Claude Desktop、Cursor、Goose、Gemini CLI）会同时加载多个 MCP 服务器。一个用户可能同时运行着文件系统服务器、Postgres 服务器和 GitHub 服务器。客户端的职责：

1. 派生每个服务器。
2. 独立地与每个服务器握手。
3. 在每个服务器上调用 `tools/list` 并将结果扁平化。
4. 当模型发出 `notes_search` 时，在合并的命名空间中查找它并路由到正确的服务器。
5. 处理来自任意服务器的通知（`tools/list_changed`）而不阻塞。
6. 在传输失败时重连。

亲手实现所有这些，正是区分“玩具”与“可用”的分水岭。官方 SDK 把这些都封装了，但心智模型必须是你自己的。

## 概念

### 子进程派生

使用 `subprocess.Popen`，并设置 `stdin=PIPE, stdout=PIPE, stderr=PIPE`。设置 `bufsize=1` 并使用文本模式进行逐行读取。每个服务器是一个进程；客户端为每个服务器持有一个 `Popen` 句柄。

### 每个服务器的会话状态

每个服务器对应一个 `Session` 对象，持有：

- `process` —— Popen 句柄。
- `capabilities` —— 服务器在 `initialize` 时声明的内容。
- `tools` —— 最后一次的 `tools/list` 结果。
- `pending` —— 请求 id 到等待响应的 promise/future 的映射。

请求本质上是异步的；发送到服务器 A 的 `tools/call` 在服务器 B 调用进行到一半时，不能阻塞它。要么使用带队列的线程，要么使用 asyncio。

### 合并的命名空间

当客户端看到聚合后的工具列表时，名称可能会冲突。两个服务器可能都暴露 `search`。客户端有三个选择：

1. **按服务器名加前缀。** `notes/search`、`files/search`。清晰但难看。
2. **静默先到先得。** 后面服务器的 `search` 会覆盖前面的。有风险；会隐藏冲突。
3. **冲突拒绝。** 拒绝加载第二个服务器；通知用户。对安全敏感的宿主最安全。

Claude Desktop 使用按服务器加前缀。Cursor 使用带明确错误的冲突拒绝。VS Code MCP 也采用按服务器加前缀。

### 路由

合并之后，一个分发表将 `tool_name -> session` 映射起来。模型按名称发出调用；客户端找到对应的会话，将一条 `tools/call` 消息写入该服务器的 stdin，然后等待响应。

### 采样回调

如果服务器在 `initialize` 时声明了 `sampling` 能力，它可能会发送 `sampling/createMessage`，请求客户端运行其 LLM。客户端必须：

1. 阻塞对该服务器的进一步请求，直到采样解决；或者，如果其实现支持并发，则流水线化处理。
2. 调用其 LLM 提供方。
3. 将响应发送回服务器。

第 11 课会端到端地讲解采样。本课为完整性起见对其做桩处理。

### 通知处理

`notifications/tools/list_changed` 意味着重新调用 `tools/list`。`notifications/resources/updated` 意味着如果该资源正在使用，则重新读取它。通知不能产生响应 —— 不要试图对它们进行 ack。

一个常见的客户端 bug：在通知滞留于流中时，让读取循环阻塞在 `tools/call` 上。使用一个后台读取线程，将每条消息推入队列；主线程出队并分发。

### 重连

传输可能失败：服务器崩溃、操作系统杀死进程、stdio 管道断裂。客户端检测到 stdout 上的 EOF，并将该会话视为已死。选项：

- 静默重启服务器并重新握手。对纯只读服务器没问题。
- 将故障暴露给用户。对带有用户可见会话的有状态服务器没问题。

Phase 13 · 09 讲解 Streamable HTTP 的重连语义；stdio 更简单。

### Keepalive 与会话 id

Streamable HTTP 使用 `Mcp-Session-Id` 头。stdio 没有会话 id —— 进程身份就是会话。Keepalive ping 是可选的；stdio 管道不会因不活动而断裂。

## 动手用

`code/main.py` 将三个模拟的 MCP 服务器派生为子进程，与每个握手，合并它们的工具列表，并将工具调用路由到正确的那个。这些“服务器”实际上是运行玩具响应器的其他 Python 进程（没有真实 LLM）。运行它以观察：

- 三次初始化，每次都有各自的能力集。
- 三个 `tools/list` 结果合并成一个 7 工具的命名空间。
- 一个基于工具名称的路由决策。
- 通过命名空间前缀避免的一次冲突。

需要关注的地方：

- `Session` 数据类干净地持有每个服务器的状态。
- 后台读取线程在不阻塞主线程的情况下将 stdout 上的每一行出队。
- 分发表是一个简单的 `dict[str, Session]`。
- 冲突处理是显式的：当两个服务器声明了相同的名称时，后者被加上前缀重命名。

## 交付

本课产出 `outputs/skill-mcp-client-harness.md`。给定一个 MCP 服务器的声明式列表（名称、命令、参数），该技能会产出一个 harness，将它们派生出来、合并工具列表，并交付一个带冲突解决的路由函数。

## 练习

1. 运行 `code/main.py` 并观察服务器派生日志。用 SIGTERM 杀死其中一个模拟服务器进程，观察客户端如何检测到 EOF 并将该会话标记为已死。

2. 实现命名空间前缀。当两个服务器都暴露 `search` 时，将第二个重命名为 `<server>/search`。更新分发表并验证工具调用能正确路由。

3. 为服务器重启添加一个连接池式的退避：在连续失败时指数退避，上限为 30 秒，在三次失败后向用户发出一个通知。

4. 勾勒一个支持 100 个并发 MCP 服务器的客户端。什么数据结构替代了简单的分发 dict？（提示：用于前缀命名空间的 trie，外加一个每服务器工具数量的指标。）

5. 将客户端移植到官方 MCP Python SDK。该 SDK 封装了 `stdio_client` 和 `ClientSession`。代码应当从 ~200 行缩减到 ~40 行，同时保留多服务器路由。

## 关键术语

| 术语 | 人们怎么说 | 它实际的含义 |
|------|----------------|------------------------|
| MCP client | “智能体宿主” | 派生服务器并编排工具调用的进程 |
| Session | “每个服务器的状态” | 能力、工具列表，以及待处理请求的记账 |
| Merged namespace | “一个工具列表” | 跨所有活动服务器的扁平工具名称集合 |
| Namespace collision | “两个服务器同一个工具” | 客户端必须对重复项加前缀、拒绝或先到先得 |
| Routing | “这次调用归谁？” | 从工具名称到拥有它的服务器的分发 |
| Background reader | “非阻塞的 stdout” | 将服务器 stdout 排空到队列中的线程或任务 |
| Sampling callback | “LLM 即服务” | 客户端对来自服务器的 `sampling/createMessage` 的处理器 |
| `notifications/*_changed` | “原语发生了变更” | 客户端必须重新发现或重新读取的信号 |
| Reconnection policy | “当服务器死亡时” | 传输失败时的重启语义 |
| Stdio session | “进程 = 会话” | 没有会话 id；子进程的生命周期就是会话 |

## 延伸阅读

- [Model Context Protocol — Client spec](https://modelcontextprotocol.io/specification/2025-11-25/client) —— 规范的客户端行为
- [MCP — Quickstart client guide](https://modelcontextprotocol.io/quickstart/client) —— 使用 Python SDK 的 hello-world 客户端教程
- [MCP Python SDK — client module](https://github.com/modelcontextprotocol/python-sdk) —— 参考 `ClientSession` 和 `stdio_client`
- [MCP TypeScript SDK — Client](https://github.com/modelcontextprotocol/typescript-sdk) —— TS 的对应实现
- [VS Code — MCP in extensions](https://code.visualstudio.com/api/extension-guides/ai/mcp) —— VS Code 如何在单个编辑器宿主中复用多个 MCP 服务器
