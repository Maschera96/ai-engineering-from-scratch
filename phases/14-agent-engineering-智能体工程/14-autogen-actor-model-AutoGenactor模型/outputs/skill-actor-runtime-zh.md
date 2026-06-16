---
name: actor-runtime-zh
description: 构建一个 AutoGen v0.4 形态的 actor 运行时，具备私有状态、每个 actor 独立收件箱、仅消息式 IPC、故障隔离以及死信队列。
version: 1.0.0
phase: 14
lesson: 14
tags: [autogen, actor-model, messaging, fault-isolation, dead-letter]
---

给定一个多智能体任务，产出一个 actor 运行时以及所需的智能体 actor。

产出：

1. 一个 `Message` 类型，包含 `sender`、`recipient`、`topic`、`body`、`mid`。
2. 一个 `Actor` 基类，包含 `receive(message, runtime)`。Actor 状态是私有的。
3. 一个 `Runtime`，包含共享队列、`send()`、`run_until_idle()` 以及一个死信队列。处理器中的异常进入 DLQ；不要向上传播。
4. 一个拓扑辅助器：RoundRobin（固定轮换）、Selector（由 LLM 挑选下一个）或自定义广播。
5. 每条消息的可观测性钩子：按照第 23 课发出带有 `gen_ai.agent.name` 和 `gen_ai.operation.name` 的 OTel span。

硬性拒绝：

- 同步消息传递，即阻塞发送方直到接收方返回。那是 v0.2 模型；它破坏了故障隔离。
- 跨 actor 的共享可变状态。Actor 通过消息读取状态，或者根本不读取。
- 一个会向上传播处理器异常的运行时。故障应归入 DLQ；让其他 actor 继续运行。

拒绝规则：

- 如果任务只有两个 actor 且是固定的来回往返，拒绝 actor 框架，并建议改用提示链（第 12 课）。当存在 >=3 个 actor 或异步并发时，actor 才值得其成本。
- 如果用户为了“更易调试”而想要“同步模式”，拒绝。建议改用日志记录 + 追踪（第 23 课）。
- 如果领域严格属于请求/响应且只有单个专家，建议改用路由（第 12 课）而非 actor 团队。

输出：`message.py`、`actor.py`、`runtime.py`、`teams.py`、`README.md`，其中说明 DLQ 策略、拓扑选择，以及 OTel span 是如何接入的。最后以“接下来读什么”收尾，如果 actor 进行协商则指向第 25 课（多智能体辩论），如果需要追踪则指向第 23 课（OTel），如果你想要面向未来的运行时则指向 Microsoft Agent Framework。
