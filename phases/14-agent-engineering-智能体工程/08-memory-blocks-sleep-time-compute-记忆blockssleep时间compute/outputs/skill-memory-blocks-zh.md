---
name: memory-blocks-zh
description: 生成一个 Letta 形态的三层记忆系统（核心 block、recall、archival），并将 sleep-time 整合智能体放在关键路径之外。
version: 1.0.0
phase: 14
lesson: 08
tags: [memory, letta, blocks, sleep-time, consolidation]
---

给定一个目标运行时、一个主模型以及一个（可能更强的）sleep-time 模型，产出一个具有明确 block 类型和异步整合的三层记忆系统。

产出：

1. 带有 `label`、`value`、`limit`、`description`、`version`、`history` 的 `Block` 类型。每次写入都会递增 version 并记录旧值。暴露 `near_limit(threshold=0.8)`。
2. 一个 `BlockStore`，至少包含三个默认 block：`human`（关于用户的事实）、`persona`（智能体的自我认知）和 `task`（当前范围）。允许用户自定义 block。
3. 一个 `Recall` 存储 —— 按会话分页的轮次日志。每一轮自动写入。达到上限时尾部会被淘汰，但仍可检索。
4. 一个 `Archival` 存储 —— 至少两种后端（向量、KV）。插入返回记录 id。出现矛盾时作废而非删除。
5. 一个 `PrimaryAgent`，处理每一轮并且只发出原始写入。关键路径上不做摘要。
6. 一个 `SleepTimeAgent`，在轮次之间运行：对超过阈值的 block 做摘要，作废被矛盾的 archival 记录，将 `learned_context` 写入共享 block。

硬性拒绝项：

- 任何在面向用户的轮次中同步运行的记忆操作，直接查找除外。摘要、整合、作废都属于 sleep-time 流程。
- 出现矛盾时删除 archival 记录。应作废，以保留可审计的历史。
- 在没有审核步骤的情况下写入 Persona 或 Safety block。这些 block 会全局性地塑造行为；静默写入会掩盖 bug。

拒绝规则：

- 如果运行时无法在多个会话之间持久化 block，则拒绝交付一个被描述为“记忆”的产品。应降低相应的宣称。
- 如果 sleep-time 智能体没有 trace 输出，则拒绝。静默整合是一个调试盲区。
- 如果用户要求“不做作废，始终信任最新写入”，那么对于任何历史性陈述很重要的领域（合规、医疗、法律），都应拒绝。

输出：每个组件一个文件，外加一个 `README.md`，其中说明默认 block、sleep-time 节奏以及矛盾解决策略。结尾以“接下来读什么”指向：如果智能体需要在记忆之上进行图推理，则指向第 09 课；如果产品需要在记忆操作上有 OTel span，则指向第 23 课。
