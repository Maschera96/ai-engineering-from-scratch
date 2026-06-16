<<<BEGIN_MARKDOWN>>>
# Claude Agent SDK：子智能体与会话存储

> Claude Agent SDK是Claude Code harness 的库形态。内置工具、用于上下文隔离的子智能体、hooks、W3C trace 传播、会话存储对等能力。Claude Managed Agents 是面向长时间异步工作的托管替代方案。

**类型：** 学习 + 构建
**语言：** Python（标准库）
**前置要求：** Phase 14 · 01（智能体循环）、Phase 14 · 10（技能库）
**时长：** 约 75 分钟

## 学习目标

- 解释 Anthropic Client SDK（原始 API）与 Claude Agent SDK（harness 形态）之间的区别。
- 描述子智能体——并行化与上下文隔离——以及何时应当使用它们。
- 说出 Python SDK 的会话存储接口（`append`、`load`、`list_sessions`、`delete`、`list_subkeys`）以及 `--session-mirror` 的作用。
- 用标准库实现一个 harness，包含内置工具、带隔离上下文的子智能体派生、生命周期 hooks，以及会话存储。

## 问题

原始 LLM API 只能让你完成一次往返。生产级智能体需要工具执行、MCP 服务器、生命周期 hooks、子智能体派生、会话持久化、trace 传播。Claude Agent SDK 把这套形态作为一个库提供——就是 Claude Code 所用的同一个 harness，向自定义智能体开放。

## 概念

### Client SDK vs Agent SDK

- **Client SDK（`anthropic`）。** 原始 Messages API。循环、工具、状态都由你自己负责。
- **Agent SDK（`claude-agent-sdk`）。** 内置工具执行、MCP 连接、hooks、子智能体派生、会话存储。把 Claude Code 的循环做成一个库。

### 内置工具

该 SDK 开箱即带 10 多个工具：文件读写、shell、grep、glob、网络抓取等等。自定义工具通过标准的 tool-schem接口注册。

### 子智能体

Anthropic 文档记载的两个用途：

1。**并行化。** 并发运行相互独立的工作。"为这 20 个模块每一个找到对应的测试文件"就是 20 个并行的子智能体任务。
2。**上下文隔离。** 子智能体使用它们自己的上下文窗口；只有结果会返回给编排者。编排者的预算得以保留。

Python SDK 近期新增：`list_subagents()`、`get_subagent_messages()`，用于读取子智能体的对话记录。

### 会话存储

与 TypeScript 的协议对等：

- `append(session_id，message)` —— 添加一个回合。
- `load(session_id)` —— 恢复对话。
- `list_sessions()` —— 枚举。
- `delete(session_id)` —— 级联删除子智能体会话。
- `list_subkeys(session_id)` —— 列出子智能体的键。

`--session-mirror`（CLI 标志）在对话记录流式输出时，把它镜像到一个外部文件，便于调试。

### Hooks

你可以注册的生命周期 hooks：

- `PreToolUse`、`PostToolUse` —— 对工具调用进行拦截或审计。
- `SessionStart`、`SessionEnd` —— 建立与拆除。
- `UserPromptSubmit` —— 在模型看到用户输入之前对其进行处理。
- `PreCompact` —— 在上下文压缩之前运行。
- `Stop` —— 智能体退出时清理。
- `Notification` —— 旁路告警。

Hooks 正是 pro-workflow（Phase 14 课程参考）及类似系统添加横切行为的方式。

### W3C trace context

调用方上活跃的 OTel span 会通过 W3C trace context 头传播进 CLI 子进程。整个多进程的 trace 会在你的后端中显示为一条 trace。

### Claude Managed Agents

托管的替代方案（bet头 `managed-agents-2026-04-01`）。长时间运行的异步工作、内置 prompt caching、内置压缩。用控制权换取托管的基础设施。

### 这个模式何处会出错

- **子智能体过度派生。** 为 100 个微小任务派生 100 个子智能体。开销占据主导。应改为批处理。
- **Hook 膨胀。** 每个团队都加 hooks；启动时间暴涨。每季度审查一次 hooks。
- **会话膨胀。** 会话不断积累；体积增长。使用 `list_sessions` + 过期策略。

## 动手构建

`code/main.py` 用标准库实现了该 SDK 的形态：

- `Tool`、`ToolRegistry`，内置 `read_file`、`write_file`、`list_dir`。
- `Subagent` —— 私有上下文、隔离运行、返回结果。
- `SessionStore` —— append、load、list、delete、list_subkeys。
- `Hooks` —— `pre_tool_use`、`post_tool_use`、`session_start`、`session_end`。
- 一个演示：主智能体并行派生 3 个子智能体（每个都隔离），聚合结果，持久化会话。

运行它：

```
python3 code/main.py
```

trace 展示了子智能体的上下文隔离（编排者的上下文大小保持有界）、hook 执行，以及会话持久化。

## 使用它

- **Claude Agent SDK** 适用于想要 Claude Code harness 形态的 Claude 优先产品。
- **Claude Managed Agents** 适用于托管的长时间异步工作。
- **OpenAI Agents SDK**（第 16 课）适用于 OpenAI 优先的对应场景。
- **LangGraph + 自定义工具** 适用于你想要图形态状态机的情况。

## 交付它

`outputs/skill-claude-agent-scaffold.md` 脚手架式生成一个 Claude Agent SDK 应用，包含子智能体、hooks、会话存储、MCP 服务器接入，以及 W3C trace 传播。

## 练习

1。添加一个子智能体派生器，把 20 个任务分批为每组 5 个并行子智能体。测量编排者的上下文大小，并与每任务一个子智能体的方式对比。
2。实现一个 `PreToolUse` hook，对 `write_file` 调用限流（每个会话每分钟 5 次）。追踪其行为。
3。接入 `list_subkeys` 来渲染一棵子智能体树。深层嵌套看起来是什么样？
4。把这个玩具实现移植到真正的 `claude-agent-sdk` Python 包上。工具注册有什么变化？
5。阅读 Claude Managed Agents 文档。什么时候你会从自托管切换到托管？

## 关键术语

| 术语 | 人们怎么说 | 它实际是什么 |
|------|----------------|------------------------|
| Agent SDK | "Claude Code 作为一个库" | Harness 形态：工具、MCP、hooks、子智能体、会话存储 |
| 子智能体 | "子级智能体" | 独立的上下文、自己的预算；结果向上冒泡 |
| 会话存储 | "对话数据库" | 持久化、加载、列举、删除回合，并级联到子智能体 |
| Hook | "生命周期回调" | 工具前/后、会话、提示提交、压缩、停止 |
| W3C trace context | "跨进程 trace" | 父 span 传播进 CLI 子进程 |
| Managed Agents | "托管 harness" | Anthropic 托管的长时间异步工作 |
| `--session-mirror` | "对话记录镜像" | 在会话回合流式输出时把它们写入外部文件 |
| MCP 服务器 | "工具面" | 接入智能体的外部工具/资源来源 |

## 延伸阅读

- [Claude Agent SDK overview](https://platform.claude.com/docs/en/agent-sdk/overview) —— Claude Code 的库形态
- [Anthropic，构建ing agents withClaude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk) —— 生产模式
- [Claude Managed Agents overview](https://platform.claude.com/docs/en/managed-agents/overview) —— 托管替代方案
- [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/) —— 对应方案

<<<END_MARKDOWN>>>
