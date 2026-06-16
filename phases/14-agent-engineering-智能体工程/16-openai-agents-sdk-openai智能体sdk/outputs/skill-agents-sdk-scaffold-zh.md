---
name: agents-sdk-scaffold-zh
description: 用分诊智能体、handoffs、输入/输出/工具护栏、会话存储和追踪处理器，搭建一个 OpenAI Agents SDK 应用脚手架。
version: 1.0.0
phase: 14
lesson: 16
tags: [openai, agents-sdk, handoffs, guardrails, tracing, session]
---

给定一个产品领域和一组专家智能体，搭建一个 OpenAI Agents SDK 应用脚手架。

产出：

1. 为每个专家创建一个 `Agent`，再加一个只有 handoffs（没有领域工具）的 `triage` 智能体。
2. 为每个领域工具创建一个 `FunctionTool`，带有类型化的输入模式、清晰的描述（告诉模型何时使用它）以及执行沙箱。
3. 从 triage 到每个专家的 `Handoff`。验证工具名称遵循 `transfer_to_<agent>` 约定。
4. 为 PII、策略、范围设置 `InputGuardrail`。默认使用并行模式，除非护栏 LLM 相对于主模型较大——此时使用阻塞模式。
5. 为长度、PII、策略设置 `OutputGuardrail`。在生产环境中，对安全关键的输出始终使用阻塞模式。
6. 对触及网络或文件系统的函数工具，设置每个工具的护栏。
7. `Session` 存储（默认 SQLite；生产环境用 Redis）。
8. `add_trace_processor` 将 span 接入你的后端，与 OpenAI 的追踪 UI 并行使用。

硬性拒绝：

- 带有领域工具的分诊智能体。Triage 只做 handoffs；混入工具会稀释路由器的决策。
- 会修改输入/输出的护栏。护栏只做批准或拒绝——它们不重写内容。
- 静默的 handoff 循环。要求设置跳数计数器（默认上限 3）。

拒绝规则：

- 如果用户想要“不要护栏，只管快速推进”，对任何触及付费用户或 PII 的产品，予以拒绝。
- 如果产品只有 2 个专家，建议改用带直接分类器的 `Agents` 路由（第 12 课），而非 triage+handoffs——token 成本更低。
- 如果生产环境禁用了追踪，拒绝上线。没有追踪，多步骤故障无法调试。

输出：`agents.py`、`tools.py`、`guardrails.py`、`app.py`、`README.md`，其中说明分诊智能体的理由、护栏模式、追踪处理器和会话后端。最后以“下一步阅读”收尾，指向第 23 课（OTel GenAI）、第 24 课（可观测性后端），或第 17 课的 Claude Agent SDK 迁移。
