# Self-Refine 与 CRITIC：迭代式输出改进

> Self-Refine（Madaan 等，2023）让一个 LLM 扮演三种角色——生成、反馈、精炼——在循环中运行。平均增益：在 7 个任务上提升 +20 绝对值。CRITIC（Gou 等，2023）通过把验证路由到外部工具来强化反馈环节。到 2026 年，这一模式已在每个框架中落地，称为"评估器-优化器"（Anthropic）或护栏循环（OpenAI Agents SDK）。

**类型：** 构建
**语言：** Python（标准库）
**前置要求：** Phase 14 · 01（智能体循环）、Phase 14 · 03（Reflexion）
**时长：** ~60 分钟

## 学习目标

- 说出 Self-Refine 的三个提示（生成、反馈、精炼），并解释为什么历史对精炼提示很重要。
- 解释 CRITIC 的关键洞见：LLM 在缺乏外部支撑时进行自我验证并不可靠。
- 用 stdlib 实现一个带历史和可选外部验证器的 Self-Refine 循环。
- 把这一模式映射到 Anthropic 的"评估器-优化器"工作流和 OpenAI Agents SDK 的输出护栏。

## 问题

一个智能体产出了一个几乎正确的答案。也许某行代码有语法错误。也许摘要太长。也许计划漏掉了一个边界情况。你想要的是：智能体评论自己的输出，然后修正它。

Self-Refine 表明这件事用单个模型就能做到，无需训练数据，无需 RL。但有一个陷阱：LLM 在硬性事实上的自我验证很差。CRITIC 点明了修复方案——把验证环节路由到外部工具（搜索、代码解释器、计算器、测试运行器）。

这两篇论文共同定义了 2026 年迭代改进的默认范式：生成、验证（尽可能借助外部）、精炼，在验证器通过时停止。

## 概念

### Self-Refine（Madaan 等，NeurIPS 2023）

一个 LLM，三种角色：

```
generate(task)            -> output_0
feedback(task，output_0)  -> critique_0
refine(task，output_0，critique_0，history) -> output_1
feedback(task，output_1)  -> critique_1
refine(task，output_1，critique_1，history) -> output_2
...
stop when feedback says "no issues" or budget exhausted.
```

关键细节：`refine` 能看到完整历史——所有先前的输出和评论——因此它不会重复犯错。论文对此做了消融实验：去掉历史，质量会急剧下降。

要点：在 7 个任务（数学、代码、缩写、对话）上平均提升 +20 绝对值，包括 GPT-4。无需训练，无需外部工具，单个模型。

### CRITIC（Gou 等，arXiv:2305.11738，v4，2024 年 2 月）

Self-Refine 的弱点：反馈环节是 LLM 在给自己打分。对于事实性断言，这并不可靠（幻觉对产生它的模型往往看起来很有说服力）。CRITIC 把 `feedback(task，output)` 替换为 `verify(task，output，tools)`，其中 `tools` 包括：

- 用于事实性断言的搜索引擎。
- 用于代码正确性的代码解释器。
- 用于算术的计算器。
- 领域专用验证器（单元测试、类型检查器、linter）。

验证器产出一份基于工具结果的结构化评论。精炼器随后以这份评论为条件。

要点：CRITIC 在事实性任务上优于 Self-Refine，因为评论有支撑。在没有外部验证器的任务上（创意写作、格式化），CRITIC 退化为 Self-Refine。

### 停止条件

两种常见形态：

1。**验证器通过。** 外部测试返回成功。在可用时优先采用（单元测试、类型检查器、护栏断言）。
2。**未提出反馈。** 模型说"输出没问题"。更便宜但不可靠；需搭配最大迭代次数上限。

2026 年默认做法：把两者结合。"如果验证器通过 OR 模型说没问题 AND 迭代次数 >= 2 OR 迭代次数 >= max_iterations，则停止。"

### 评估器-优化器（Anthropic，2024）

Anthropic 2024 年 12 月的文章把它列为五种工作流模式之一。两种角色：

- 评估器：给输出打分并产出评论。
- 优化器：根据评论修订输出。

循环直到评估器通过。这就是 Anthropic 框架下的 Self-Refine/CRITIC。Anthropic 补充的关键工程细节：评估器和优化器的提示应当有实质性差异，这样模型才不会只是走过场盖章。

### OpenAI Agents SDK 输出护栏

OpenAI Agents SDK 以"输出护栏"形态提供这一模式。护栏是一个在智能体最终输出上运行的验证器。如果护栏触发（抛出 `OutputGuardrailTripwireTriggered`），输出会被拒绝，智能体可以重试。护栏可以调用工具（CRITIC 风格），也可以是纯函数（Self-Refine 风格）。

### 2026 年的陷阱

- **盖章式循环。** 同一个模型用同样的提示风格做生成和评论，会收敛到"我看挺好"。请使用结构上不同的提示，或用一个更小更便宜的模型做评论。
- **过度精炼。** 每一次精炼都会增加延迟和 token。预算 1-3 次；超过之后，升级到人工审查。
- **在琐碎任务上用 CRITIC。** 如果没有外部验证器，CRITIC 会退化为 Self-Refine；不要为一个桩验证器付出延迟代价。

## 动手构建

`code/main.py` 在一个玩具任务上实现了 Self-Refine 和 CRITIC：给定一个主题，产出一份简短的项目符号列表。验证器检查格式（3 个项目符号，每个不超过 60 个字符）。CRITIC 额外加入一个外部"事实验证器"，对已知幻觉进行惩罚。

组件：

- `generate` —— 脚本化的生产者。
- `feedback` —— LLM 风格的自我评论。
- `verify_external` —— CRITIC 风格的有支撑验证器。
- `refine` —— 根据历史重写输出。
- 停止条件 —— 验证器通过或最多 4 次迭代。

运行它：

```
python3 code/main.py
```

对比 Self-Refine 与 CRITIC 的运行结果。CRITIC 捕捉到了一个 Self-Refine 漏掉的事实错误，因为外部验证器拥有自我评论器所没有的支撑。

## 使用它

Anthropic 的评估器-优化器就是用 Claude 友好的语言表述的这一模式。OpenAI Agents SDK 的输出护栏是 CRITIC 形态的（护栏可以调用工具）。LangGraph 提供了一个反思节点，读起来就像 Self-Refine。Google 的 Gemini 2.5 Computer Use 加入了一个逐步安全评估器，它是 CRITIC 的一个变体：每个动作在提交前都会被验证。

## 交付它

`outputs/skill-refine-loop.md` 根据任务形态、验证器可用性和迭代预算来配置一个评估器-优化器循环。它会输出生成器、评估器/验证器和优化器的提示，外加一份停止策略。

## 练习

1。用 max_iterations=1 运行这个玩具任务。CRITIC 还有帮助吗？
2。把外部验证器替换为一个有噪声的版本（随机 30% 的假阳性）。循环会怎么做？这正是 2026 年大多数护栏栈的现实。
3。实现一个"生成器-评论器分用不同模型"的变体：大模型生成，小模型评论。它能胜过同模型方案吗？
4。阅读 CRITIC 第 3 节（arXiv:2305.11738 v4）。说出三类验证工具，并为每一类各举一个例子。
5。把 OpenAI Agents SDK 的 `output_guardrails` 映射到 CRITIC 的验证器角色。SDK 哪里做错了，哪里做对了？

## 关键术语

| 术语 | 人们怎么说 | 它实际是什么 |
|------|----------------|------------------------|
| Self-Refine | "会自我修正的 LLM" | 单个模型中的生成 -> 反馈 -> 精炼循环，带历史 |
| CRITIC | "工具支撑的验证" | 用外部验证器（搜索、代码、计算器、测试）替换反馈 |
| 评估器-优化器 | "Anthropic 工作流模式" | 两种角色——评估器打分，优化器修订——循环至收敛 |
| 输出护栏 | "事后检查" | OpenAI Agents SDK 在智能体产出输出后运行的验证器 |
| 验证环节 | "评论阶段" | 承重的决策：有支撑还是自评 |
| 精炼历史 | "模型已经试过什么" | 先前的输出 + 评论，前置到精炼提示中；去掉它质量就崩溃 |
| 盖章式循环 | "自我认同失败" | 同提示评论返回"看着挺好"；用结构上不同的提示修复 |
| 停止条件 | "收敛测试" | 验证器通过 OR 无反馈 AND 迭代上限；绝不用单一条件 |

## 延伸阅读

- [Madaan et al.，Self-Refine (arXiv:2303.17651)](https://arxiv.org/abs/2303.17651) —— 经典论文
- [Gou et al.，CRITIC (arXiv:2305.11738)](https://arxiv.org/abs/2305.11738) —— 工具支撑的验证
- [Anthropic，构建ing Effective Agents](https://www.anthropic.com/research/building-effective-agents) —— 评估器-优化器工作流模式
- [OpenAI Agents SDK docs](https://openai.github.io/openai-agents-python/) —— 作为 CRITIC 形态验证器的输出护栏
