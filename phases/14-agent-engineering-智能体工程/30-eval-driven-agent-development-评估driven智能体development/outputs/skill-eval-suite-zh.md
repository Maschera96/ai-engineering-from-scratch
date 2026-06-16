---
name: eval-suite-zh
description: 构建三层评估套件（静态基准、自定义离线、线上生产），包含 evaluator-optimizer 循环与 CI 门禁。
version: 1.0.0
phase: 14
lesson: 30
tags: [evaluation, ci, regression, benchmarks, llm-judge]
---

给定一个智能体产品，构建一套接入 CI 的三层评估套件。

产出：

1. **静态基准层** —— 至少一个相关基准（代码用 SWE-bench Verified，工具使用用 BFCL V4，Web 用 WebArena，桌面用 OSWorld，通用任务用 GAIA）。始终同时报告经审计（+-audited）的分数。
2. **自定义离线层** —— 至少一个在领域特定维度上打分的 LLM-judge 评分量表（事实性、语气、范围、拒答质量）。至少一个基于执行的用例，在智能体运行后探查实际状态。至少一个带有黄金路径的基于轨迹的用例。
3. **线上评估层** —— 会话回放、由护栏触发的告警、通过 OTel GenAI spans 进行的每步成本/延迟追踪（第 23 课）。
4. **evaluator-optimizer 运行器** —— 将智能体包裹在 propose / judge / refine 中，并设置轮次上限。
5. **CI 门禁** —— 相对基线回归 >=5% 时让构建失败。持续追踪基线随时间的变化。
6. **用例映射** —— Phase 14 各课中的每条护栏和每条习得规则都至少对应一个用例。

硬性拒绝：

- 没有基线的评估套件。没有参考就无法检测回归。
- 在事实性任务上使用没有外部依据支撑的 LLM-judge。必须使用 CRITIC 模式（第 05 课）。
- 没有固定种子或快照状态的不稳定用例。误报会侵蚀团队对评估的信任。

拒绝规则：

- 如果用户只想要“走通顺路径就行”，拒绝。每种失败模式（第 26 课）都应有一个用例。
- 如果产品已面向付费用户，而用户想要“不设 CI 门禁”，拒绝。否则评估漂移将不可见。
- 如果用户想要“全部用 LLM-judge”，在事实性和合规任务上拒绝。那里必须使用基于执行或编程式的评估器。

输出：`cases/benchmarks/`、`cases/custom/`、`cases/online/`、`runner.py`、`ci_gate.py`、`README.md`，其中说明评分量表、基线以及 Phase 14 映射表。以“接下来读什么”结尾，指向第 24 课（可观测性）、第 26 课（失败模式）或第 23 课（OTel）作为底层支撑。
