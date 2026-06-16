---
name: state-graph-zh
description: 构建一个 LangGraph 形态的状态机，具备带类型的状态、条件边、逐节点检查点以及持久化恢复能力。
version: 1.0.0
phase: 14
lesson: 13
tags: [langgraph, state-machine, durable, checkpointing, human-in-the-loop]
---

给定一个目标运行时、一个状态结构、一组节点函数以及一个检查点后端，生成一个有状态的智能体图。

产出：

1. 一个带类型的 `State`（dict 或 Pydantic）。为每个字段编写文档。节点读取状态；它们返回更新。
2. 一个 `StateGraph`，包含 `add_node`、`add_edge`、`add_conditional_edges`、`set_entry`，外加 `START`/`END` 哨兵。
3. 一个 `Checkpointer` 接口，包含 `save(session_id, node, state)` 和 `load_latest(session_id)`。默认使用 SQLite；允许 Postgres/Redis/自定义。
4. 一个 `Runner`，逐步执行图，在每个节点之后序列化状态，捕获用于 human-in-the-loop 的 `PausedAtNode`，并支持带可选 `state_override` 的 `resume_from`。
5. 三个拓扑辅助器：supervisor（中心路由器）、swarm（共享工具的交接）、hierarchical（子图）。

硬性拒绝：

- 没有显式随机种子或 wall-clock 捕获的非确定性节点。恢复假定在给定输入状态时节点输出是可复现的。
- 仅保存“摘要”状态的检查点。请序列化完整状态，否则恢复会失效。
- 每条边都是条件边的图。优先使用偶尔带分支的线性链。

拒绝规则：

- 如果用户请求一个没有持久化的状态图，拒绝。其全部意义在于持久化恢复；如果你不需要恢复，请使用第 12 课中的工作流模式。
- 如果用户请求“仅在成功时检查点”，拒绝。失败时也需要状态——那正是调试的起点。
- 如果图的节点超过约 30 个，拒绝扁平布局并要求使用嵌套子图。扁平的 30 节点图无法被审查。

输出：`state.py`、`graph.py`、`checkpointer.py`、`runner.py`、`README.md`，说明状态模式、检查点选择以及恢复语义。最后以“接下来读什么”收尾，指向第 14 课的 actor 模型替代方案、第 16 课的交接/护栏层，或第 23 课中针对图步骤的 OTel span。
