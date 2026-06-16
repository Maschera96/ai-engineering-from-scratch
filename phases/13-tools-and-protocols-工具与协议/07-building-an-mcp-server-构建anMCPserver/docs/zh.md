# 构建一个 MCP Server —— Python + TypeScript SDK

> 大多数 MCP 教程只展示 stdio 的 hello-world。一个真正的 server 会同时暴露 tools、resources 和 prompts，处理能力协商，发出结构化错误，并在不同 SDK 之间保持一致的行为。本课会端到端地构建一个 notes server：基于标准库的 stdio 传输、JSON-RPC 分发、三种 server 原语，以及一种纯函数风格——当你将来升级时，它可以直接套进 Python SDK 的 FastMCP 或 TypeScript SDK。

**类型：** Build
**语言：** Python（标准库，stdio MCP server）
**前置要求：** Phase 13 · 06（MCP 基础）
**时长：** ~75 分钟

## 学习目标

- 实现 `initialize`、`tools/list`、`tools/call`、`resources/list`、`resources/read`、`prompts/list` 和 `prompts/get` 方法。
- 编写一个分发循环，从 stdin 读取 JSON-RPC 消息并将响应写入 stdout。
- 按照 JSON-RPC 2.0 规范以及 MCP 的额外错误码发出结构化错误响应。
- 将标准库实现升级到 FastMCP（Python SDK）或 TypeScript SDK，而无需重写工具逻辑。

## 问题

在你能使用远程传输（Phase 13 · 09）或鉴权层（Phase 13 · 16）之前，你需要一个干净的本地 server。本地意味着 stdio：server 由客户端作为子进程启动，消息以换行符分隔的形式在 stdin/stdout 上流动。

2025-11-25 规范规定，stdio 消息被编码为 JSON 对象，并带有显式的 `\n` 分隔符。这里没有 SSE；SSE 是旧的远程模式，将在 2026 年年中被移除（Atlassian 的 Rovo MCP server 于 2026 年 6 月 30 日弃用它；Keboola 于 2026 年 4 月 1 日弃用）。对于 stdio，每行一个 JSON 对象就是整个线缆格式。

notes server 是一个很好的形态，因为它能演练全部三种 server 原语。Tools 执行变更（`notes_create`）。Resources 暴露数据（`notes://{id}`）。Prompts 提供模板（`review_note`）。本课的这种形态可以推广到任何领域。

## 概念

### 分发循环

```
loop:
  line = stdin.readline()
  msg = json.loads(line)
  if has id:
    handle request -> write response
  else:
    handle notification -> no response
```

三条规则：

- 不要向 stdout 打印任何不是 JSON-RPC 信封的内容。调试日志写到 stderr。
- 每个请求都必须用携带相同 `id` 的响应来匹配。
- 通知（notification）绝不能被响应。

### 实现 `initialize`

```python
def initialize(params):
    return {
        "protocolVersion": "2025-11-25",
        "capabilities": {
            "tools": {"listChanged": True},
            "resources": {"listChanged": True, "subscribe": False},
            "prompts": {"listChanged": False},
        },
        "serverInfo": {"name": "notes", "version": "1.0.0"},
    }
```

只声明你支持的能力。客户端依赖这个能力集合来对功能进行门控。

### 实现 `tools/list` 和 `tools/call`

`tools/list` 返回 `{tools: [...]}`，其中每一项都带有 `name`、`description`、`inputSchema`。`tools/call` 接收 `{name, arguments}`，并返回 `{content: [blocks], isError: bool}`。

内容块（content block）是带类型的。最常见的有：

```json
{"type": "text", "text": "Found 2 notes"}
{"type": "resource", "resource": {"uri": "notes://14", "text": "..."}}
{"type": "image", "data": "<base64>", "mimeType": "image/png"}
```

工具错误有两种形态。协议级错误（未知方法、错误参数）是 JSON-RPC 错误。工具级错误（调用合法但工具执行失败）则以 `{content: [...], isError: true}` 的形式返回。这能让模型在其上下文中看到这个失败。

### 实现 resources

Resources 在设计上就是只读的。`resources/list` 返回一份清单；`resources/read` 返回内容。URI 可以是 `file://...`、`http://...`，或像 `notes://` 这样的自定义 scheme。

当你把数据作为 resource 而不是 tool 暴露时：

- 模型不会“调用”它；客户端可以在用户请求时把它注入到上下文中。
- 订阅（subscription）让 server 能在 resource 发生变化时推送更新（Phase 13 · 10）。
- Phase 13 · 14 用 `ui://` 将其扩展为交互式 resource。

### 实现 prompts

Prompts 是带有命名参数的模板。宿主（host）将它们呈现为斜杠命令。一个 `review_note` prompt 可能接收一个 `note_id` 参数，并生成一份多消息的 prompt 模板，供客户端喂给它的模型。

### stdio 传输的细节

- 换行符分隔的 JSON。没有长度前缀的分帧。
- 不要缓冲。每次写入后调用 `sys.stdout.flush()`。
- 客户端控制生命周期。当 stdin 关闭（EOF）时，干净地退出。
- 不要静默处理 SIGPIPE；记录日志并退出。

### 注解（Annotations）

每个 tool 都可以携带描述安全属性的 `annotations`：

- `readOnlyHint: true` —— 纯读取，重试是安全的。
- `destructiveHint: true` —— 不可逆的副作用；客户端应当确认。
- `idempotentHint: true` —— 相同输入产生相同输出。
- `openWorldHint: true` —— 与外部系统交互。

客户端用这些来决定 UX（确认对话框、状态指示器）和路由（Phase 13 · 17）。

### 升级路径

`code/main.py` 中的标准库 server 大约 180 行。FastMCP（Python）将相同的逻辑收缩为装饰器风格：

```python
from fastmcp import FastMCP
app = FastMCP("notes")

@app.tool()
def notes_search(query: str, limit: int = 10) -> list[dict]:
    ...
```

TypeScript SDK 有等价的形态。当你准备好时，升级路径是即插即用的；其中的概念（能力、分发、内容块）都是一样的。

## 动手用

`code/main.py` 是一个完整的、仅基于标准库、运行在 stdio 之上的 notes MCP server。它处理 `initialize`、`tools/list`、针对三个 tool（`notes_list`、`notes_search`、`notes_create`）的 `tools/call`、针对每条笔记的 `resources/list` 和 `resources/read`，以及一个 `review_note` prompt。你可以通过管道传入 JSON-RPC 消息来驱动它：

```
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' | python main.py
```

重点关注：

- 分发器是一个以方法名为键的 `dict[str, Callable]`。
- 每个 tool 执行器返回的是一个内容块列表，而不是裸字符串。
- 当执行器抛出异常时，会设置 `isError: true`。

## 交付

本课产出 `outputs/skill-mcp-server-scaffolder.md`。给定一个领域（notes、tickets、files、database），该 skill 会脚手架出一个 MCP server，并带有正确的 tools / resources / prompts 拆分以及 SDK 升级路径。

## 练习

1. 运行 `code/main.py`，并用手工构建的 JSON-RPC 消息来驱动它。演练 `notes_create`，然后用 `resources/read` 取回新建的笔记。

2. 添加一个带 `annotations: {destructiveHint: true}` 的 `notes_delete` tool。验证客户端会弹出确认对话框（这需要一个真实的宿主；Claude Desktop 可以）。

3. 实现 `resources/subscribe`，使得每当某条笔记被修改时，server 都会推送 `notifications/resources/updated`。再加一个 keepalive 任务。

4. 把 server 移植到 FastMCP。Python 文件应当缩小到 80 行以内。线缆行为必须完全一致；用相同的 JSON-RPC 测试夹具来验证。

5. 阅读规范的 `server/tools` 部分，找出一个本课 server 未实现的 tool 定义字段。（提示：有好几个；选一个并把它加上。）

## 关键术语

| 术语 | 人们怎么说 | 它实际指什么 |
|------|----------------|------------------------|
| MCP server | “暴露 tools 的那个东西” | 在 stdio 或 HTTP 上讲 MCP JSON-RPC 的进程 |
| stdio transport | “子进程模型” | server 由客户端启动；通过 stdin/stdout 通信 |
| Dispatcher（分发器） | “方法路由器” | 从 JSON-RPC 方法名到处理函数的映射 |
| Content block（内容块） | “工具结果片段” | tool 响应的 `content` 数组中带类型的元素 |
| `isError` | “工具级失败” | 表示该 tool 失败；与 JSON-RPC 错误区分开 |
| Annotations（注解） | “安全提示” | readOnly / destructive / idempotent / openWorld 标志 |
| FastMCP | “Python SDK” | 构建在 MCP 协议之上的、基于装饰器的更高层框架 |
| Resource URI | “可寻址数据” | `file://`、`db://` 或标识某个 resource 的自定义 scheme |
| Prompt template（prompt 模板） | “斜杠命令简报” | server 提供的、带参数槽位、供宿主 UI 使用的模板 |
| Capability declaration（能力声明） | “功能开关” | 在 `initialize` 中按原语声明的标志 |

## 延伸阅读

- [Model Context Protocol — Python SDK](https://github.com/modelcontextprotocol/python-sdk) —— 参考性的 Python 实现
- [Model Context Protocol — TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) —— 平行的 TS 实现
- [FastMCP — server framework](https://gofastmcp.com/) —— 面向 MCP server 的装饰器风格 Python API
- [MCP — Quickstart server guide](https://modelcontextprotocol.io/quickstart/server) —— 使用任一 SDK 的端到端教程
- [MCP — Server tools spec](https://modelcontextprotocol.io/specification/2025-11-25/server/tools) —— tools/* 消息的完整参考
