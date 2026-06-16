# 异步任务（SEP-1686）—— 即刻调用、稍后获取，应对长时间运行的工作

> 真实的智能体工作往往需要数分钟到数小时：CI 运行、深度研究综合、批量导出。同步工具调用会断开连接、超时，或阻塞 UI。SEP-1686 于 2025-11-25 合并，新增了 Tasks 原语：任何请求都可以被增强为一个任务，其结果可以稍后获取，或通过状态通知进行流式传输。漂移风险提示：Tasks 在 2026 上半年仍属实验性质；SDK 接口仍在围绕规范设计中。

**类型：** Build
**语言：** Python（stdlib，异步任务状态机）
**前置条件：** Phase 13 · 07（MCP server）、Phase 13 · 09（transports）
**时长：** 约 75 分钟

## 学习目标

- 识别何时应将一个工具从同步提升为任务增强（服务器端工作 >30 秒）。
- 走通任务生命周期：`working` → `input_required` → `completed` / `failed` / `cancelled`。
- 持久化任务状态，使崩溃不会丢失进行中的工作。
- 正确地轮询 `tasks/status` 并获取 `tasks/result`。

## 问题所在

一个 `generate_report` 工具运行着耗时数分钟的提取流水线。在同步模型下的选项有：

1. 将连接保持三分钟。远程传输会断开它；客户端会超时；UI 会冻结。
2. 立即返回一个占位符；要求客户端轮询自定义端点。这破坏了 MCP 的统一性。
3. 发射后不管（fire-and-forget）；没有结果。

没有一个是好的。SEP-1686 增加了第四种：任务增强。任何请求（通常是 `tools/call`）都可以被标记为任务。服务器立即返回一个任务 id。客户端轮询 `tasks/status`，并在完成时获取 `tasks/result`。服务器端状态可在重启后存续。

## 核心概念

### 任务增强

通过设置 `params._meta.task.required: true`（或 `optional: true`，由服务器决定），一个请求就变成了任务。服务器立即响应：

```json
{
  "jsonrpc": "2.0", "id": 1,
  "result": {
    "_meta": {
      "task": {
        "id": "tsk_9f7b...",
        "state": "working",
        "ttl": 900000
      }
    }
  }
}
```

`ttl` 是服务器保留状态的承诺；超过 ttl 后，任务结果将被丢弃。

### 逐工具选择启用

工具注解可以声明任务支持：

- `taskSupport: "forbidden"` —— 此工具始终同步运行。适合快速工具。
- `taskSupport: "optional"` —— 客户端可以请求任务增强。
- `taskSupport: "required"` —— 客户端必须使用任务增强。

`generate_report` 工具应为 `required`。`notes_search` 工具应为 `forbidden`。

### 状态

```
working  -> input_required -> working  (loop via elicitation)
working  -> completed
working  -> failed
working  -> cancelled
```

状态机是仅追加的：一旦进入 `completed`、`failed` 或 `cancelled`，任务即为终态。

### 方法

- `tasks/status {taskId}` —— 返回当前状态和进度提示。
- `tasks/result {taskId}` —— 阻塞，或在尚未完成时返回 404。
- `tasks/cancel {taskId}` —— 幂等；终态会被忽略。
- `tasks/list` —— 可选；枚举活跃及最近完成的任务。

### 流式状态变化

当服务器支持时，客户端可以订阅状态通知：

```
server -> notifications/tasks/updated {taskId, state, progress?}
```

采用流式而非轮询的客户端能获得更好的用户体验。轮询作为最小接口始终被支持。

### 持久化状态

规范要求声明了任务支持的服务器持久化状态。崩溃不应在 ttl 内丢失已完成的结果。存储方案从 SQLite 到 Redis，再到文件系统皆可。Lesson 13 的练习框架使用文件系统。

### 取消语义

`tasks/cancel` 是幂等的。如果任务正在执行中，服务器会尝试停止它（检查执行器协作式取消）。如果已是终态，该请求是空操作（no-op）。

### 崩溃恢复

当服务器进程重启时：

1. 加载所有已持久化的任务状态。
2. 将进程已死亡的任何 `working` 任务标记为 `failed`，错误为 `CRASH_RECOVERY`。
3. 在 ttl 期间保留 `completed` / `failed` / `cancelled`。

### 异步任务与 sampling 结合

一个任务本身可以调用 `sampling/createMessage`。长时间运行的研究任务正是这样工作的：服务器的任务线程按需对客户端的模型进行采样，同时客户端的 UI 将任务显示为 `working`，并定期更新进度。

### 为何这是实验性的

SEP-1686 于 2025-11-25 发布，但更宏观的路线图指出了三个未决问题：持久化订阅原语、子任务（父子任务关系），以及结果 TTL 的标准化。预计该规范会在 2026 年持续演进。生产代码应仅将 Tasks 视为在常见场景下稳定，并防范未来 SDK 在子任务方面的变化。

## 动手用起来

`code/main.py` 实现了一个持久化任务存储（基于文件系统），以及一个在后台线程中运行的 `generate_report` 工具。客户端调用该工具，立即获得一个任务 id，在 worker 更新进度期间轮询 `tasks/status`，并在完成时获取 `tasks/result`。取消功能可用；崩溃恢复通过杀死 worker 线程并重新加载状态来模拟。

需要关注的地方：

- 任务状态 JSON 被持久化到 `/tmp/lesson-13-tasks/<id>.json`。
- worker 线程更新 `progress` 字段；轮询会显示其推进。
- 客户端发起的取消会设置一个事件；worker 检查后提前退出。
- “崩溃”后的状态重新加载会将进行中的任务标记为 `failed`，错误为 `CRASH_RECOVERY`。

## 交付成果

本课程产出 `outputs/skill-task-store-designer.md`。给定一个长时间运行的工具（研究、构建、导出），该 skill 会设计任务存储（状态结构、ttl、持久性），选择正确的 taskSupport 标志，并勾勒出进度通知。

## 练习

1. 运行 `code/main.py`。启动一个 `generate_report` 任务，轮询状态，然后获取结果。

2. 在运行中途添加一次 `tasks/cancel` 调用。验证 worker 遵从它，并且状态变为 `cancelled`。

3. 模拟崩溃恢复：杀死 worker 线程，重启加载器，并观察 `CRASH_RECOVERY` 失败模式。

4. 将存储扩展为 SQLite。持久性收益相同；查询选项随之打开（列出来自 session X 的所有任务）。

5. 阅读 2026 年的 MCP 路线图帖子。识别出未来一年内最有可能影响 SDK API 设计的那个与 Tasks 相关的未决问题。

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| Task | “长时间运行的工具调用” | 用 `_meta.task` 增强、用于异步执行的请求 |
| SEP-1686 | “Tasks 规范” | 于 2025-11-25 加入 Tasks 的 Spec Evolution Proposal |
| `_meta.task` | “任务信封” | 包含 id、state、ttl 的逐请求元数据 |
| taskSupport | “工具标志” | 每个工具的 `forbidden` / `optional` / `required` |
| `tasks/status` | “轮询方法” | 获取当前状态和可选的进度提示 |
| `tasks/result` | “获取结果” | 返回已完成的负载，或在尚未完成时返回 404 |
| `tasks/cancel` | “停下它” | 幂等的取消请求 |
| ttl | “保留预算” | 服务器承诺保留任务状态的毫秒数 |
| `notifications/tasks/updated` | “状态推送” | 服务器发起的状态变化事件 |
| 持久化存储 | “崩溃安全的状态” | 文件系统 / SQLite / Redis 持久化层 |

## 延伸阅读

- [MCP — GitHub SEP-1686 issue](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1686) —— 原始提案与完整讨论
- [WorkOS — MCP async tasks for AI agent workflows](https://workos.com/blog/mcp-async-tasks-ai-agent-workflows) —— 带原理说明的设计走查
- [DeepWiki — MCP task system and async operations](https://deepwiki.com/modelcontextprotocol/modelcontextprotocol/2.7-task-system-and-async-operations) —— 机制与状态机
- [FastMCP — Tasks](https://gofastmcp.com/servers/tasks) —— SDK 层面的任务实现模式
- [MCP blog — 2026 roadmap](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/) —— 未决问题与 2026 年优先事项，包括子任务
