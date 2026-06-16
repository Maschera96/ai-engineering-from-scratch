---
name: crew-or-flow-zh
description: 为给定任务选择 CrewAI Crew 或 Flow，并搭建最小实现。
version: 1.0.0
phase: 14
lesson: 15
tags: [crewai, crews, flows, multi-agent, role-based]
---

给定一个任务描述，选择 Crew（自主）或 Flow（确定性），然后搭建脚手架。

决策：

1. 任务是否有 SLA、合规或确定性重放要求？-> Flow。
2. 任务是否属于探索性质（研究、初稿、头脑风暴）？-> Crew。
3. 任务是否有 4 个以上专家且由 LLM 决定顺序？-> Hierarchical Crew。
4. 任务是否有不超过 3 个专家且顺序固定？-> Sequential Crew 或 Flow——优先选 Flow。

对于 Crew，产出：

1. 智能体定义：role、goal、backstory（精炼，不超过 200 词）、tools。
2. 任务定义：description、expected_output、agent。
3. 使用正确 Process（Sequential | Hierarchical）的 Crew。
4. 一个测试框架，在样本输入上运行 Crew 并检查是否产出了预期的 expected_outputs。

对于 Flow，产出：

1. `@start` 入口函数。
2. 构成 DAG 的 `@listen(topic)` 步骤。
3. 显式的事件 topic；不要使用魔法式广播。
4. 一个重放框架：给定一个 kickoff 载荷，确定性地重新运行。

硬性拒绝：

- 没有 backstory 的 Crew。backstory 是承重结构。
- 没有显式 topic 名称的 Flow。“隐式链式调用”会破坏审计目的。
- 只有 2 个专家的 Hierarchical Crew。管理者的开销不值得这份成本。

拒绝规则：

- 如果用户在仅限生产环境的合规任务上要求使用 Crew，拒绝并迁移到 Flow。
- 如果用户在开放式研究任务上要求使用 Flow，拒绝并迁移到 Crew。
- 如果 backstory 超过 200 词，拒绝并要求精简。上下文预算是有限的。

输出：`agents.py`、`tasks.py`、`crew.py` 或 `flow.py`，外加包含决策依据的 `README.md`。结尾给出“接下来读什么”，指向第 24 课（Langfuse/AgentOps）以了解可观测性，或者若 Flow 需要持久化恢复语义则指向第 13 课。
