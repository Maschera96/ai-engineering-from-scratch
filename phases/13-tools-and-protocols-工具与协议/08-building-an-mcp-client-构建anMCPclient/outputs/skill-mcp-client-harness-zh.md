---
name: mcp-client-harness-zh
description: 给定一个声明式的 MCP 服务器列表（name、command、args），搭建一个具备握手、命名空间合并与路由的多服务器客户端。
version: 1.0.0
phase: 13
lesson: 08
tags: [mcp, client, multi-server, routing, namespace]
---

给定一组要运行的 MCP 服务器配置，生成一个客户端框架（harness），它会启动每个服务器、与每个服务器握手、将它们的工具列表合并到一个命名空间中，并将每次调用路由到拥有该工具的服务器。

产出：

1. 服务器配置解析器。映射 `name -> {command, args, env}`。校验命令在 path 上存在。
2. 启动计划。使用 subprocess.Popen，配合 stdin/stdout/stderr 管道、`bufsize=1`、文本模式。每个服务器一个后台读取线程。
3. 握手流水线。对每个会话：发送 `initialize`，等待响应，持久化 capabilities，发送 `notifications/initialized`。
4. 命名空间合并。选择一种冲突策略：`prefix-on-collision`（默认）、`reject-on-collision` 或 `silent-overwrite`（禁止使用）。在启动时打印合并后的工具列表。
5. 路由函数。`client.call(canonical_name, arguments)` 查找拥有该工具的会话并写出一条 `tools/call` 消息。通过 pending-request 表中的 future 等待 id 匹配的响应。

硬性拒绝：
- 任何不为每个服务器各自启动独立进程的 harness。进程内多路复用会破坏隔离模型。
- 任何将 `silent-overwrite` 作为默认冲突策略的 harness。存在安全风险。
- 任何在 stdout 读取上阻塞主线程的 harness。通知会被卡住。

拒绝规则：
- 如果某个服务器的命令不可信（不在固定的允许列表中），拒绝启动，并路由到 Phase 13 · 15 进行安全检查。
- 如果用户在没有理由的情况下配置了超过 10 个服务器，发出警告并建议使用网关（Phase 13 · 17）。
- 如果被要求在此处理 OAuth，拒绝并路由到 Phase 13 · 16。

输出：一个完整的 client-harness Python 文件（约 150 行），包含 Session、合并逻辑、路由，以及一个会逐一调用每个已配置服务器的主循环。以一行总结收尾，说明所选的冲突策略以及合并后的工具数量。
