# AutoGen v0.4：Actor 模型与智能体框架

> AutoGen v0.4（Microsoft Research，2025 年 1 月）围绕 actor 模型重新设计了智能体编排。异步消息交换、事件驱动的智能体、按 actor 的故障隔离、天然的并发。该框架现已进入维护模式，由 Microsoft Agent Framework（2025 年 10 月公开预览）作为继任者。

**类型：** 学习 + 构建
**语言：** Python（标准库）
**前置条件：** Phase 14 · 01（智能体循环）、Phase 14 · 12（工作流模式）
**时间：** 约 75 分钟

## 学习目标

- 描述 actor 模型：智能体作为 actor，消息作为唯一的 IPC，每个 actor 的故障隔离。
- 说出 AutoGen v0.4 的三个 API 层——Core、AgentChat、Extensions——以及各自的用途。
- 解释为何将消息投递与处理解耦能带来故障隔离和天然并发。
- 用 Python 实现一个 stdlib actor 运行时，并将一个双智能体代码评审流程移植到其上。

## 问题

大多数智能体框架是同步的：一个智能体生产，一个智能体消费，都在一个调用栈中。故障会让整个栈崩溃。并发是后期硬加上去的。分布式则需要重写。

AutoGen v0.4 的答案：actor 模型。每个智能体都是一个拥有私有收件箱的 actor。消息是唯一的交互方式。运行时将投递与处理解耦。故障被隔离到单个 actor。并发是原生的。分布式只是不同的传输方式而已。

## 概念

### Actor

一个 actor 拥有：

- 私有状态（绝不从外部直接触碰）。
- 收件箱（消息队列）。
- 处理器：`receive(message) -> effects`，其中 effects 可以是"回复"、"发送给其他 actor"、"生成新 actor"、"更新状态"、"停止自身"。

两个 actor 不能共享内存。它们只能互相发送消息。

### AutoGen v0.4 中的三个 API 层

1。**Core。** 低层 actor 框架。`AgentRuntime`、`Agent`、`Message`、`Topic`。异步消息交换、事件驱动。
2。**AgentChat。** 任务驱动的高层 API（替代 v0.2 的 ConversableAgent）。`AssistantAgent`、`UserProxyAgent`、`RoundRobinGroupChat`、`SelectorGroupChat`。
3。**Extensions。** 各类集成——OpenAI、Anthropic、Azure、工具、记忆。

### 解耦为何重要

在 v0.2 模型中，同步调用 `agent_a.chat(agent_b)` 会阻塞 agent_直到 agent_b 返回。在 v0.4 中，`send(agent_b，msg)` 把消息放入 agent_b 的收件箱后即返回。运行时稍后再投递。这带来三个结果：

- **故障隔离。** Agent B 崩溃不会让 Agent A 崩溃——运行时在 B 的处理器中捕获故障，并决定如何处理（记录日志、重试、死信）。
- **天然并发。** 多条消息同时在途；actor 并发处理各自的收件箱。
- **面向分布式。** 无论 actor 是在进程内还是在另一台主机上，收件箱 + 传输都是同一套抽象。

### 拓扑

- **RoundRobinGroupChat。** 智能体按固定轮转依次发言。
- **SelectorGroupChat。** 一个选择器智能体根据对话上下文挑选下一个发言者。
- **Magentic-One。** 用于网页浏览、代码执行、文件处理的参考多智能体团队。构建于 AgentChat 之上。

### 可观测性

内置 OpenTelemetry 支持。每条消息都发出一个 span；工具调用按 2026 年 OTel GenAI 语义约定（第 23 课）携带 `gen_ai.*` 属性。

### 状态：维护模式

2026 年初：AutoGen v0.7.x 对研究和原型开发而言是稳定的。Microsoft 已将活跃开发转向 Microsoft Agent Framework（2025 年 10 月 1 日公开预览；1.0 GA 目标定在 2026 年第一季度末）。AutoGen 的模式可以干净地向前移植——actor 模型才是经久不衰的核心思想。

## 动手构建

`code/main.py` 实现了一个 stdlib actor 运行时：

- `Message`——带 `sender`、`recipient`、`topic`、`body` 的类型化负载。
- `Actor`——抽象类，带 `receive(message，runtime)`。
- `Runtime`——带共享队列、投递、故障隔离的事件循环。
- 一个双 actor 演示：`ReviewerAgent` 评审代码，`ChecklistAgent` 执行检查清单；它们互发消息直到达成共识。

运行它：

```
python3 code/main.py
```

trace 展示了消息投递、一个 actor 中的模拟故障（不会让另一个崩溃），以及在共享结论上的收敛。

## 应用

- **AutoGen v0.4/v0.7**（维护中）——对研究、原型开发、多智能体模式而言稳定。
- **Microsoft Agent Framework**（公开预览）——向前的路径；同样的 actor 模型思想，配以焕新的 API。
- **LangGraph swarm 拓扑**（第 13 课）——通过共享工具交接实现的类似模式。
- **自定义 actor 运行时**——当你需要特定传输（NATS、RabbitMQ、gRPC）时。

## 交付

`outputs/skill-actor-runtime.md` 为给定的多智能体任务生成一个最小 actor 运行时外加一个团队模板（RoundRobin 或 Selector）。

## 练习

1。添加一个死信队列：当处理器抛出异常时，将失败的消息暂存以供人工检查。在你的玩具示例里 DLQ 被命中的频率有多高？
2。实现 `SelectorGroupChat`：一个选择器 actor 根据对话状态挑选谁来处理下一条消息。
3。添加分布式传输：把进程内队列换成一个 JSON-over-HTTP 服务器，让 actor 能在独立进程中运行。
4。为每条消息接入一个 OTel span（或一个空操作替身）。按第 23 课发出 `gen_ai.agent.name`、`gen_ai.operation.name`。
5。阅读 AutoGen v0.4 的架构文章。把你的玩具示例移植到真正的 `autogen_core` API。你跳过了哪些在生产中很重要的部分？

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| Actor | "智能体" | 私有状态 + 收件箱 + 处理器；无共享内存 |
| Message | "事件" | 类型化负载；actor 交互的唯一方式 |
| Inbox | "邮箱" | 每个 actor 的待处理消息队列 |
| Runtime | "智能体宿主" | 路由消息并隔离故障的事件循环 |
| Topic | "频道" | actor 之间命名的发布-订阅路由 |
| Fault isolation | "任它崩溃" | 一个 actor 故障不会让其他 actor 崩溃 |
| RoundRobinGroupChat | "固定轮转团队" | 智能体按顺序轮流发言 |
| SelectorGroupChat | "上下文路由团队" | 选择器挑选下一个发言者 |
| Magentic-One | "参考团队" | 用于网页 + 代码 + 文件的多智能体小队 |

## 延伸阅读

- [AutoGen v0.4，Microsoft Research](https://www.microsoft.com/en-us/research/articles/autogen-v0-4-reimagining-the-foundation-of-agentic-ai-for-scale-extensibility-and-robustness/) —— 这篇重新设计的文章
- [LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview) —— 图形态的替代方案
- [OpenTelemetry GenAI semantic conventions](https://opentelemetry.io/docs/规范s/semconv/gen-ai/) —— AutoGen 默认发出的 span
