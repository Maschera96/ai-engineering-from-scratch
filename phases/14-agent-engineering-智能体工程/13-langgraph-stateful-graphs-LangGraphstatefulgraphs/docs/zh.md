# LangGraph：有状态图与持久化执行

> LangGraph是2026 年低层级有状态编排的参考标准。智能体即状态机；节点是函数；边是转移；状态是不可变的，并在每一步之后被检查点（checkpoint）保存。可以从任何故障点精确地恢复，从中断处继续。

**类型：** 学习 + 构建
**语言：** Python（标准库）
**前置要求：** Phase 14 · 01（智能体循环），Phase 14 · 12（工作流模式）
**时间：** ~75 分钟

## 学习目标

- 描述 LangGraph 的核心模型：具有不可变状态、函数节点、条件边和步后检查点的状态机。
- 说出文档强调的四项能力：持久化执行、流式传输、人在回路（human-in-the-loop）、全面记忆。
- 解释 LangGraph 支持的三种编排拓扑：supervisor（监督者）、peer-to-peer（对等 / swarm）、hierarchical（分层 / 嵌套子图）。
- 实现一个 stdlib 状态图，包含不可变状态、条件边以及检查点/恢复循环。

## 问题

智能体与工作流共享一个问题：当一次 40 步的运行在第 38 步失败时，你希望从第 38 步恢复，而不是从头开始。二等公民式的状态模型让运维人员只能围绕一个假设每次都是全新运行的库去拼凑各种重试逻辑。

LangGraph 的设计答案是：状态是一等的类型化对象，变更是显式的，检查点在每个节点之后持久化。恢复就是一次 `load_state(session_id)` 调用。

## 概念

### 图

一个图由以下部分定义：

- **状态类型（State type）。** 一个类型化字典（或 Pydantic 模型），每个节点都会读取并变更它。
- **节点（Nodes）。** 纯函数 `(state) -> state_update`。返回后更新会被合并进状态。
- **边（Edges）。** 节点之间的条件转移或直接转移。
- **入口与出口（Entry and exit）。** `START` 和 `END` 哨兵节点标记边界。

示例：一个具有 `classify`、`refund`、`bug`、`sales`、`done` 节点的智能体——一个以图形式表达的路由工作流。

### 持久化执行

每个节点返回后，运行时会序列化状态并将其写入检查点器（checkpointer，SQLite、Postgres、Redis、自定义）。当在第 N 步失败时，运行时可以执行 `resume(session_id)`，并带着精确的状态从第 N+1 步继续。

LangGraph 文档明确强调了这一点重要的生产用户：Klarna、Uber、J.P。Morgan。其论点不在于图的形状本身；而在于图的形状加上检查点使得恢复变得廉价。

### 流式传输

每个节点都可以产出部分输出。图会向调用方流式传输每个节点的增量（per-node-delta）事件，从而让 UI 随着图的运行而更新。

### 人在回路

在节点之间检查并修改状态。实现方式：在某个关键节点之前暂停，将状态呈现给人类，接受修改，然后恢复。检查点器让这件事变得容易，因为状态已经被序列化了。

### 记忆

短期（在一次运行之内——状态中的对话历史）和长期（跨多次运行——通过检查点器加上一个独立的长期存储来持久化）。LangGraph 通过工具与外部记忆系统（Mem0、自定义）集成。

### 三种拓扑

1。**Supervisor（监督者）。** 中心路由 LLM 将任务分派给专家子智能体。在 `langgraph-supervisor` 中使用 `create_supervisor()`（不过 LangChain 团队在 2026 年建议直接通过工具调用来实现这一点，以获得更强的上下文控制）。
2。**Swarm / peer-to-peer（蜂群 / 对等）。** 智能体通过共享的工具表面直接交接（hand off）。没有中心路由器。
3。**Hierarchical（分层）。** 由监督者管理子监督者，以嵌套子图（nested subgraphs）的形式实现。

### 这种模式哪里会出错

- **检查点太小。** 只对对话轮次做检查点会导致工具状态和记忆写入无法恢复。完整状态必须被序列化。
- **非确定性节点。** 恢复假设节点输入会产生相同的状态更新。随机种子、墙上时钟时间、外部 API 都必须被捕获。
- **过度使用条件边。** 一个每条边都是条件边的图，是一台无法被推理的状态机。优先采用偶尔分支的线性链。

## 动手构建

`code/main.py` 实现了一个 stdlib 有状态图：

- `State` —— 一个带有 `messages`、`step`、`route`、`output`、`human_approval` 的类型化字典。
- `Node` —— 接收状态并返回更新字典的可调用对象。
- `StateGraph` —— 节点 + 边 + 条件边 + 运行 + 恢复。
- `SQLiteCheckpointer`（内存中的假实现）—— 在每个节点之后序列化状态；`load(session_id)` 进行恢复。
- 一个演示图：classify -> branch(refund / bug / sales) -> human gate（人工闸门）-> send。

运行它：

```
python3 code/main.py
```

该跟踪展示了第一次运行在人工闸门处失败、持久化、然后恢复并产生最终输出的过程。

## 使用它

- **LangGraph** —— 参考标准，可用于生产。使用 `create_react_agent`、`create_supervisor`，或构建你自己的图。
- **AutoGen v0.4**（第 14 课）—— 面向高并发场景的 actor 模型替代方案。
- **Claude Agent SDK**（第 17 课）—— 带有内置会话存储的托管式 harness。
- **自定义** —— 当你需要对状态形状或检查点器后端进行精确控制时。

## 交付它

`outputs/skill-state-graph.md` 可在任何目标运行时生成一个 LangGraph 形态的状态图，并接好检查点与恢复。

## 练习

1。当分类置信度低于某个阈值时，从 `classify` 添加一条到 `end` 的条件边。在人类手动设置 `route` 之后恢复该运行。
2。把类 SQLite 的假实现替换为真正的 SQLite 检查点器。测量每步的序列化开销。
3。实现并行边：两个节点并发运行，通过自定义 reducer 合并。不可变状态在这里带来了什么？
4。阅读 `langgraph-supervisor` 参考文档。将这个玩具示例移植到 `create_supervisor`。比较跟踪形态。
5。添加流式传输：每个节点在运行时产出部分状态。在增量到达时将它们打印出来。

## 关键术语

| 术语 | 人们常说的 | 它实际的含义 |
|------|----------------|------------------------|
| State graph（状态图） | “智能体即状态机” | 类型化状态 + 节点 + 边 + reducer |
| Checkpointer（检查点器） | “持久化后端” | 在每个节点之后序列化状态；支持恢复 |
| Reducer（归约器） | “状态合并器” | 将当前状态与节点更新组合起来的函数 |
| Conditional edge（条件边） | “分支” | 由状态的函数选择的边 |
| Subgraph（子图） | “嵌套图” | 作为另一个图内部节点使用的图 |
| Durable execution（持久化执行） | “从故障恢复” | 带着精确状态从最后一个成功节点重启 |
| Supervisor（监督者） | “路由 LLM” | 面向专家子智能体的中心分派器 |
| Swarm（蜂群） | “P2P 智能体” | 智能体通过共享工具交接；没有中心路由器 |

## 延伸阅读

- [LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview) —— 参考文档
- [langgraph-supervisor reference](https://reference.langchain.com/python/langgraph/supervisor/) —— supervisor 模式 API
- [AutoGen v0.4，Microsoft Research](https://www.microsoft.com/en-us/research/articles/autogen-v0-4-reimagining-the-foundation-of-agentic-ai-for-scale-extensibility-and-robustness/) —— actor 模型替代方案
- [Claude Agent SDK overview](https://platform.claude.com/docs/en/agent-sdk/overview) —— 会话存储与子智能体
