---
name: runtime-picker-zh
description: 为给定的技术栈、延迟预算和运维形态挑选生产级智能体运行时（Agno、Mastra、LangGraph、提供商 SDK）。
version: 1.0.0
phase: 14
lesson: 18
tags: [agno, mastra, langgraph, runtime, selection]
---

给定一个技术栈、延迟预算、所需原语和运维形态，挑选一个运行时。

决策：

1. Python + FastAPI + 每秒数千个短生命周期智能体 -> **Agno**。
2. TypeScript + Next.js/Vercel + 统一的多提供商 -> **Mastra**。
3. 持久状态、显式图、失败后可恢复 -> **LangGraph**（第 13 课）。
4. 以 Claude 为先的产品，想要 Claude Code harness 的形态 -> **Claude Agent SDK**（第 17 课）。
5. 以 OpenAI 为先的产品，想要交接 + 护栏 + 追踪 -> **OpenAI Agents SDK**（第 16 课）。
6. 多智能体团队、actor 模型并发、故障隔离 -> **AutoGen v0.4** / **Microsoft Agent Framework**（第 14 课）。
7. 基于角色的协作或事件驱动的确定性工作流 -> **CrewAI** Crew 或 Flow（第 15 课）。
8. 以上都不是 -> 直接调用 API + 第 01 课的标准库循环。

产出：

- 一份简短的决策文档：技术栈、延迟目标、所需原语、观察到的取舍。
- 在所选运行时中的最小脚手架。
- 如果目前已在使用另一个运行时，则提供一份迁移计划。

硬性否决：

- 在工作负载是每个请求一次慢调用时，仅凭“性能”就选 Agno 或 Mastra。性能很少是瓶颈。
- 在 Python monorepo 中没有理由地选用 TypeScript 运行时。混合语言的智能体代码是一种运维税。
- 为无状态的短任务选用 LangGraph。checkpointer 带来的开销是简单工作流（第 12 课）所能避免的。

拒绝规则：

- 如果用户想要“全部五种运行时来做对比”，拒绝。在你自己的工作负载上做基准测试；框架厂商的基准测试只是方向性参考。
- 如果用户想自托管 Mastra 的 `ee/` 功能，拒绝并指向其许可条款。
- 如果产品需要长时间运行的异步工作（数小时到数天），拒绝自托管并路由到 Claude Managed Agents 或基于队列的架构（第 29 课）。

输出：决策文档 + 脚手架 + README。最后以“接下来读什么”收尾，指向第 24 课（可观测性）和第 29 课（生产运行时），即框架之上的运维层。
