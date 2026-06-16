---
name: claude-agent-scaffold-zh
description: 用子智能体、生命周期钩子、会话存储、MCP 服务器挂载和 W3C 追踪传播来搭建 Claude Agent SDK 应用脚手架。
version: 1.0.0
phase: 14
lesson: 17
tags: [claude-agent-sdk, subagents, hooks, session-store, mcp]
---

给定一个产品领域和一组 MCP 服务器，搭建一个 Claude Agent SDK 应用脚手架。

产出：

1. 一个主智能体定义，包含指令、内置工具访问（read_file、write_file、shell、grep、glob、web fetch）以及自定义函数工具。
2. 用于并行化和上下文隔离的子智能体生成器。当编排器原本会耗尽其上下文预算时使用。
3. 注册的生命周期钩子：用于审计的 PreToolUse + PostToolUse、用于初始化的 SessionStart、用于清理的 SessionEnd、用于规则强制执行的 UserPromptSubmit（参见 pro-workflow 模式）。
4. 会话存储（默认 SQLite），其 `list_subkeys` 接好以渲染子智能体树。
5. 用于挂载外部工具/资源界面的 MCP 服务器挂载。
6. W3C 追踪上下文传播，使来自调用方的 OTel span 能贯穿 CLI 延续下去。

硬性拒绝：

- 为单个工具的任务生成子智能体。子智能体用于并行化或上下文隔离，而不是为了"一次 read_file 调用"。
- 包含同步昂贵工作的钩子。钩子应在微秒到毫秒级别。长时间工作应放在子智能体中。
- 没有级联删除策略的会话存储。孤立的子智能体会话会让存储膨胀。

拒绝规则：

- 如果产品需要长时间运行的异步工作（数小时到数天），拒绝使用自托管 SDK，并路由到 Claude Managed Agents。
- 如果用户要求 `--session-mirror` 到共享位置，拒绝。会话记录包含 PII；应镜像到按用户加密的存储。
- 如果智能体在没有工具使用的情况下依赖原始 LLM 流式输出来支撑 UX，拒绝使用 Agent SDK，并直接推荐 Client SDK。

输出：`agent.py`、`tools.py`、`hooks.py`、`session.py`、`README.md`，说明子智能体策略、钩子注册表、会话后端、MCP 挂载以及 OTel 接线。最后以"接下来读什么"结尾，指向第 22 课的语音交接、第 23 课的 OTel span 归因，或者如果产品需要生产运行时形态则指向第 18 课。
