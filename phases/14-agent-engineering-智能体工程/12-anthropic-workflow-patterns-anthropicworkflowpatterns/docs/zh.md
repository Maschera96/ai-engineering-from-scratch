# Anthropic 的工作流模式：简单优于复杂

> Schluntz 和 Zhang（Anthropic，2024 年 12 月）区分了工作流（预定义路径）与智能体（动态使用工具）。五种工作流模式覆盖了大多数场景。从直接调用 API 开始。只有当步骤无法预测时才引入智能体。

**类型：** 学习 + 构建
**语言：** Python（标准库）
**前置知识：** 阶段 14 · 01（智能体循环）
**时长：** 约 60 分钟

## 学习目标

- 说出 Anthropic 的五种工作流模式：prompt chaining、routing、parallelization、orchestrator-workers、evaluator-optimizer。
- 解释智能体与工作流的区别，以及各自的工程成本。
- 判断何时该选择工作流而非智能体（反之亦然）。
- 用标准库针对一个脚本化 LLM 实现全部五种模式。

## 问题

团队常常为只需一次函数调用就能解决的问题动用多智能体框架。代价是真实存在的：框架增加了一层层结构，遮蔽了提示词，隐藏了控制流，并诱发过早的复杂化。Schluntz 和 Zhang 在 2024 年 12 月发表的文章是业界引用最多的反思之作：从简单开始，只有当复杂性配得上它的代价时才引入它。

## 概念

### 工作流与智能体

- **工作流。** 通过预定义代码路径来编排 LLM 和工具。图由工程师掌控。
- **智能体。** LLM 动态地指挥自己的工具并自行决定步骤。图由模型掌控。

两者各有其用武之地。工作流更便宜、更快、更易调试。智能体能解锁开放式问题，但让失败模式更难推理。

### 增强型 LLM

五种模式的共同基础：一个 LLM 接入三种能力——搜索（检索）、工具（动作）、记忆（持久化）。任何 API 调用都可以使用这些能力。

### 五种模式

1。**Prompt chaining（提示链）。** 第 1 次调用的输出作为第 2 次调用的输入。适用于任务有清晰线性分解的情况。各步骤之间可选地加入程序化校验门。

2。**Routing（路由）。** 一个分类器 LLM 决定调用哪个下游 LLM 或工具。适用于类别不同的输入需要不同处理方式的情况（一线支持 vs 退款 vs 故障 vs 销售）。

3。**Parallelization（并行化）。** 并发运行 N 次 LLM 调用，再聚合结果。两种形态：分段（不同的片段）和投票（相同的提示，运行 N 次，多数表决/综合）。

4。**Orchestrator-workers（编排者-工作者）。** 一个编排者 LLM 动态决定运行哪些工作者（也是 LLM），并综合它们的输出。类似于智能体循环，但编排者不会无限循环。

5。**Evaluator-optimizer（评估者-优化者）。** 一个 LLM 提出答案，另一个 LLM 对其进行评估。迭代直至评估者通过。这是 Self-Refine（第 05 课）的泛化版本。

### 工作流胜过智能体的场景

- **可预测的任务。** 如果你能枚举出所有步骤，那就应该枚举。
- **成本受限的任务。** 工作流的步骤数量有界；智能体则可能失控膨胀。
- **合规受限的任务。** 审计人员想直接读懂这张图，而不是从执行轨迹中去推断。

### 智能体胜过工作流的场景

- **开放式研究。** 当下一步取决于上一步返回了什么时。
- **长度可变的任务。** 工作量从几分钟到几小时不等，步骤数量未知。
- **新颖领域。** 当你还不知道正确的工作流是什么时——先探索，后固化。

### 上下文工程这个伴生学科

《Effective context engineering for AI agents》（Anthropic 2025）将这门相邻学科形式化：20 万 token 的窗口是一份预算，而非一个容器。该包含什么、何时压缩、何时让上下文增长。在阶段 14 关于上下文压缩的课程中有详细讲解（在重新编号之前，本课程中位于较早的阶段 14 第 06 课）。

## 动手构建

`code/main.py` 针对一个 `ScriptedLLM` 实现了全部五种工作流模式：

- `prompt_chain(input，steps)` —— 顺序执行。
- `route(input，classifier，handlers)` —— 分类 + 分派。
- `parallel_vote(prompt，n，aggregator)` —— 运行 N 次，聚合。
- `orchestrator_workers(task，workers)` —— 编排者挑选工作者。
- `evaluator_optimizer(task，proposer，evaluator，max_iter)` —— 循环直至通过。

运行它：

```
python3 code/main.py
```

每种模式都会打印自己的执行轨迹。每种模式的总代码量约 10-15 行；而框架的代价是以数千行计的。

## 使用它

- 大多数任务直接调用 API。
- 只有当某个模式确实需要持久化状态（LangGraph）、actor 模型并发（AutoGen v0.4）或角色模板（CrewAI）时才动用框架。
- 当你想要 Claude Code 的脚手架形态但又不想从头重建时，使用 Claude Agent SDK。

## 交付它

`outputs/skill-workflow-picker.md` 会为给定的任务描述挑选合适的模式，包括决策依据，以及当工作流不够用时重构为智能体的路径。

## 练习

1。实现带置信度阈值的路由。低于阈值 -> 升级给人工。对于一线支持的用例，这个阈值应该落在哪里？
2。给 `parallel_vote` 加上超时。当某次调用卡住时会发生什么？在票数缺失的情况下你该如何聚合？
3。把 `evaluator_optimizer` 改造成一个老虎机（bandit）：跨迭代保留排名前 2 的输出，这样一个出现较晚的好结果就不会被一个出现较晚的坏结果覆盖。
4。把 prompt chaining 与 routing 结合起来：一个路由器从三条链中挑选一条。对比它与单个大提示方案的 token 成本。
5。挑一个你的生产功能。画出它的工作流图。数一数步骤。在这里智能体真的会更好吗？

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| Workflow | “预定义流程” | 由工程师掌控的 LLM 与工具调用图 |
| Agent | “自主 AI” | 由模型掌控的图；动态指挥工具 |
| Augmented LLM | “带工具的 LLM” | LLM + 搜索 + 工具 + 记忆；原子单元 |
| Prompt chaining | “顺序调用” | 第 N 次调用的输出是第 N+1 次调用的输入 |
| Routing | “分类器分派” | 挑选由哪条链/模型来处理输入 |
| Parallelization | “扇出” | N 次并发调用；按分段或投票聚合 |
| Orchestrator-workers | “调度者智能体” | 编排者 LLM 动态挑选专家 LLM |
| Evaluator-optimizer | “提议者 + 裁判” | 迭代直至评估者通过；Self-Refine 的泛化 |

## 延伸阅读

- [Anthropic，构建ing Effective Agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents) —— 五种工作流模式
- [Anthropic，Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) —— 伴生学科
- [LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview) —— 有状态图何时配得上它的代价
- [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/) —— 编排者-工作者模式的产品化实现
