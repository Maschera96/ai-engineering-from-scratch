---
name: task-store-designer-zh
description: 为长时间运行的 MCP 工具设计任务存储：状态结构、ttl、持久性、取消、崩溃恢复。
version: 1.0.0
phase: 13
lesson: 13
tags: [mcp, tasks, durable-store, long-running, sep-1686]
---

给定一个长时间运行的工具（研究、构建、导出、报告生成），设计支撑 SEP-1686 任务增强的任务存储。

产出：

1. 状态结构。最少字段：`id`、`state`、`progress`、`result`、`error`、`ttl`、`created_at`。可选：`request_meta`、`parent_task_id`（用于未来的子任务）。
2. 持久性选择。玩具级用文件系统；单进程用 SQLite；多副本用 Redis。给出理由。
3. taskSupport 标志。每个工具为 `forbidden`、`optional` 或 `required`；附一行理由。
4. 取消方案。worker 如何检查取消信号；部分进度时会发生什么。
5. 崩溃恢复。启动时的重载规则；客户端看到的 `CRASH_RECOVERY` 失败是什么样子。

硬性拒绝：
- 任何在 ttl 内丢失已完成结果的存储。
- 任何没有明确终止状态（`completed`、`failed`、`cancelled`）的任务状态。
- 任何非幂等的取消。

拒绝规则：
- 如果工具运行时间在 5 秒以内，拒绝将其提升为任务。同步更简单。
- 如果任务会产生超过 10 MB 的结果，拒绝并建议使用流式内容块。
- 如果服务器没有能够持久化状态的进程（无状态边缘函数），拒绝并建议迁移到持久化运行时。

输出：一页纸的存储设计，包含状态结构、持久性选择、taskSupport 标志、取消方案和崩溃恢复规则。最后用一行给出建议：当 SEP-1686 子任务发布时是否会影响此设计。
