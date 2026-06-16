---
name: orchestration-picker-zh
description: 为给定问题选择一种编排拓扑（supervisor、swarm、hierarchical、debate 或无），并以最小方式实现它。
version: 1.0.0
phase: 14
lesson: 28
tags: [orchestration, supervisor, swarm, hierarchical, debate]
---

给定一个产品领域和一类任务，选择最小的拓扑。

决策：

1. 1 个智能体 + 工作流模式（第 12 课）就够用了吗？-> 完全不要使用拓扑。
2. 2-4 个职责清晰的专才？-> **supervisor-worker**。
3. 延迟敏感且专才之间能够干净地交接？-> **swarm**。
4. 10 个以上专才，supervisor 的上下文预算撑不住了？-> **hierarchical**。
5. 准确性比成本更重要，多提议者 + 评判会有帮助？-> **debate**（第 25 课）。

产出：

1. 所选拓扑的脚手架。
2. swarm 上的跳数计数器；hierarchical 上的嵌套深度上限；debate 上的轮次上限。
3. 每次交接或每个步骤的可观测性挂钩（OTel GenAI span，第 23 课）。
4. 一段“为什么是这个，而不是那个”的 README 章节。

硬性拒绝：

- 把 3 次顺序的 LLM 调用称为“多智能体”。那是一条提示链。
- 没有跳数计数器的 swarm。来回弹跳是必然的。
- 每个分支最终只落到 1 个专才的 hierarchical。请扁平化。

拒绝规则：

- 如果用户想为一个单个 ReAct 循环就能处理的任务使用多智能体，拒绝并建议第 01 课。
- 如果用户想为一个 2 步任务使用 supervisor，拒绝并建议提示链（第 12 课）。
- 如果该领域有合规 / 审计要求，拒绝 swarm 并建议 supervisor 或 hierarchical。

输出：拓扑脚手架 + 含决策依据的 README。以“接下来读什么”结尾，指向第 13 课（LangGraph）了解 supervisor 实现、第 16 课（OpenAI Agents SDK）了解交接即工具，或第 25 课了解 debate 的具体细节。
