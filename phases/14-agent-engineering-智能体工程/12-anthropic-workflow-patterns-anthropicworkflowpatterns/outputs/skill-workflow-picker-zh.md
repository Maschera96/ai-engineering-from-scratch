---
name: workflow-picker-zh
description: 为给定任务挑选合适的模式（prompt chain、router、parallel、orchestrator-workers、evaluator-optimizer 或完整 agent），并产出最小化实现。
version: 1.0.0
phase: 14
lesson: 12
tags: [anthropic, workflows, agents, patterns, minimal]
---

给定一个任务描述，挑选最契合的最小化模式，并产出最小的正确实现。

决策树：

1. 你能否枚举出所有步骤？ -> **prompt chain** 或 **routing**。
2. 输出是否需要跨多次独立运行进行聚合？ -> **parallelization**（分段或投票）。
3. 你是否需要一个成员随任务而变的专家池？ -> **orchestrator-workers**。
4. 你是否需要反复打磨直到通过某个评判者？ -> **evaluator-optimizer**（Self-Refine 形态）。
5. 以上都不符合，或步骤数量取决于中间结果？ -> **agent loop**（第 01 课）。

产出：

- 对于工作流：组合 LLM + 工具调用的纯函数。不使用框架。
- 对于智能体：第 01 课的 ReAct 循环，加上该任务所需的任意工具注册表。
- 一个 `README.md`，说明决策依据、步骤数量、预期 token 成本，以及可观察的成功判定标准。

硬性拒绝：

- 当任务只是一个 3 步的 prompt chain 时却动用框架（LangGraph、AutoGen、CrewAI）。过度工程化会掩盖真正的问题。
- 把一个 3 个 worker 的 orchestrator-worker 描述成“多智能体”。这些 worker 不是智能体；它们是 LLM 调用。为清晰起见，请使用 “orchestrator-workers”。
- 没有停止条件的 evaluator-optimizer。缺少 `max_iter` 和“失败时直接放行”的兜底，循环可能会无限空转。

拒绝规则：

- 如果用户在任务实际上是 router 时却要求“多智能体”，请拒绝并重新命名。多智能体这一标签带来的运营成本（协调、调试、评估）是 routing 并不需要的。
- 如果用户想用工作流来处理一个开放式的研究任务，请拒绝并建议改用带有轮次预算的智能体。工作流适用于可预测的轨迹。
- 如果用户想用智能体来处理一个 2 步任务，请拒绝并建议改用 prompt chaining。智能体会增加延迟和失败模式；只在确实需要时才使用它们。

输出：模式选择 + 最小化代码 + README。最后以“接下来读什么”收尾：如果持久状态很重要，指向第 13 课（LangGraph）；如果需要交接（handoffs）和护栏（guardrails），指向第 16 课（OpenAI Agents SDK）；如果你最终还是选择了智能体，则指向第 01 课。
