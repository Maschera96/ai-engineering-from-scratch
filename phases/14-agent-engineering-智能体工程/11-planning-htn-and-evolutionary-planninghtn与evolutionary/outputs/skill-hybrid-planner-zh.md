---
name: hybrid-planner-zh
description: 构建混合规划器——用 ChatHTN 生成可证明可靠的计划，用 AlphaEvolve 配合机器可校验的评估器进行代码搜索——并为问题选对合适的那一个。
version: 1.0.0
phase: 14
lesson: 11
tags: [planning, htn, chathtn, alphaevolve, evolutionary-search]
---

给定一类问题（受策略约束的工作流 vs 代码优化 vs 开放式任务），选择一个规划器并产出正确的脚手架。

决策：

1. 问题是否有硬性前置条件 / 策略 / 调度约束？-> HTN（ChatHTN）。
2. 问题是否有确定性的、机器可校验的适应度函数？-> 进化式（AlphaEvolve）。
3. 都没有？-> 改用 ReAct（第 01 课）或 ReWOO（第 02 课）。

对于 HTN，产出：

1. 带有 `preconditions`、`effects_add`、`effects_remove` 的 `Operator` 类型。
2. 带有 `task`、`preconditions`、`subtasks` 的 `Method` 类型。
3. 一个规划器，它先尝试方法（method），回退到 LLM 分解，并缓存成功的 LLM 分解。
4. 一个验证步骤，拒绝引用未知 operator 或 method 的 LLM 分解。

对于进化式，产出：

1. 一个候选程序的种子种群。
2. 一个返回标量适应度的确定性评估器。
3. 一个变异算子（LLM 驱动或基于规则）。
4. 一个选择循环（保留 top-k、变异、重复），并带有提前停止。

硬性拒绝：

- ChatHTN：在未经 operator 模式验证的情况下直接应用 LLM 输出。可靠性主张会失效。
- AlphaEvolve：评估器调用 LLM 评判者（judge）。适应度必须是确定性的；LLM 评判者会引入随机噪声，循环无法从中恢复。
- 把任一模式用于开放式任务（“写一篇博客文章”）。没有评估器、没有前置条件 -> 使用 ReAct。

拒绝规则：

- 如果领域没有清晰的 operator 模式，拒绝 ChatHTN。建议 ReWOO 或纯 ReAct。
- 如果领域没有机器可校验的适应度，拒绝 AlphaEvolve。建议 Self-Refine（第 05 课）。
- 如果用户想要“规划器 + LLM 做最终决定”，拒绝。符号正确性与 LLM 探索之间的切分是承重的。

输出：`operators.py`、`methods.py`、`planner.py`（HTN）或 `evaluator.py`、`mutator.py`、`loop.py`（进化式），外加包含决策依据的 `README.md`。以“接下来读什么”结尾：如果辩论式验证适合该问题，指向第 25 课；如果该任务实际上终究是 ReWOO 形态，则指向第 02 课。
