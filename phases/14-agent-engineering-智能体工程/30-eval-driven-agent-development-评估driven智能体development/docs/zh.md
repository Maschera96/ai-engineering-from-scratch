# 评估驱动的智能体开发

> Anthropic 的指导意见："从简单的提示开始，用全面的评估来优化它们，只在需要时才加入多步骤的智能体系统。"评估不是最后一步。它是驱动 Phase 14 中其他每一个选择的外层循环。

**类型：** Learn + Build
**语言：** Python (stdlib)
**前置要求：** Phase 14 全部内容。
**时长：** 约 60 分钟

## 学习目标

- 说出三个评估层次——静态基准、自定义离线、在线生产——以及各自的用途。
- 解释 evaluator-optimizer 紧密循环。
- 描述 2026 年的最佳实践：评估与代码放在一起、在 CI 中运行、对 PR 进行门禁把关。
- 把 Phase 14 的每一节课与它所产生的评估用例联系起来。

## 问题

智能体能通过演示。它们却以演示无法预测的方式在生产环境中失败。基准回答的是"这个模型整体能力如何？"而不是"这个智能体是否在为我的产品交付正确的补丁？"答案是：在三个层次上进行评估，持续运行，并将每一道护栏和习得的规则都映射到一个评估用例上。

## 概念

### 三个评估层次

1. **静态基准** —— 用于代码的 SWE-bench Verified（第 19 课）、用于浏览 / 桌面的 WebArena/OSWorld（第 20 课）、用于通才的 GAIA（第 19 课）、用于工具使用的 BFCL V4（第 06 课）。用于跨模型对比和回归门禁。污染是真实存在的：SWE-bench+ 发现了 32.67% 的解答泄漏。务必报告 Verified / +-审计后的分数。

2. **自定义离线评估** —— 你产品的形态：
   - LLM-as-judge（Langfuse、Phoenix、Opik——第 24 课）。
   - 基于执行的（运行补丁，检查测试）。
   - 基于轨迹的（将动作序列与标准答案对比；OSWorld-Human 显示顶尖智能体是标准答案的 1.4-2.7 倍）。

3. **在线评估** —— 生产环境：
   - 会话回放（Langfuse）。
   - 护栏触发的告警（第 16、21 课）。
   - 每一步的成本 / 延迟追踪（第 23 课 OTel spans）。

### Evaluator-optimizer（Anthropic）

紧密循环：

1. 提议者生成输出。
2. 评估器进行判断。
3. 持续优化，直到评估器通过。

这是 Self-Refine（第 05 课）的泛化形式。任何你关心的智能体流程都可以包裹进 evaluator-optimizer 以提升可靠性。

### 2026 年最佳实践

- 评估与代码放在一起。
- 在每个 PR 上于 CI 中运行。
- 以评估分数对合并进行门禁把关（例如"相对于 main 的回归不超过 5%"）。
- 每一道护栏都映射到一个评估用例。
- 每一条习得的规则（Reflexion、pro-workflow learn-rule）都映射到一个失败用例。

### 把 Phase 14 串联起来

Phase 14 的每一节课都会产生评估用例：

| 课程 | 它产生的评估用例 |
|--------|------------------------|
| 01 Agent Loop | 预算耗尽、无限循环防护 |
| 02 ReWOO | 工具失败时规划器能正确重新规划 |
| 03 Reflexion | 习得的反思在重试时生效 |
| 05 Self-Refine/CRITIC | 评审通过优化后的输出 |
| 06 Tool Use | 参数强制转换有效；未知工具被拒绝 |
| 07-10 Memory | 检索引用与来源匹配；过期事实被作废 |
| 12 Workflow Patterns | 每种模式都产生正确输出 |
| 13 LangGraph | 恢复能精确重现状态 |
| 14 AutoGen Actors | DLQ 捕获崩溃的处理器 |
| 16 OpenAI Agents SDK | 护栏在正确的输入上触发 |
| 17 Claude Agent SDK | 子智能体结果返回给编排器 |
| 19-20 Benchmarks | SWE-bench Verified 分数、WebArena 成功率、OSWorld 效率 |
| 21 Computer Use | 每步安全检查捕获注入的 DOM |
| 23 OTel | spans 发出所需的属性 |
| 26 Failure Modes | 检测器标记已知失败 |
| 27 Prompt Injection | PVE 拒绝被投毒的检索结果 |
| 28 Orchestration | 监督者路由到正确的专家 |
| 29 Runtime Shapes | DLQ 处理 N% 的失败 |

如果你的评估套件为每一项都准备了用例，你就覆盖了 Phase 14。

### 评估驱动开发会在哪里失败

- **没有基线。** 没有"最近已知良好"状态的评估是无法解读的。要存储基线。
- **没有依据的 LLM 评审。** 评审也会产生幻觉。CRITIC 模式（第 05 课）——让评审依据外部工具进行判断。
- **对评估过拟合。** 为评估优化会偏离生产环境中的实际用处。要轮换用例。
- **不稳定的评估。** 非确定性用例会引发误报。固定随机种子、对状态做快照。

## 动手构建

`code/main.py` 是一个基于 stdlib 的评估测试框架：

- 带类别（benchmark、custom、online）的用例注册表。
- 一个被测的脚本化智能体。
- Evaluator-optimizer 循环：提议、判断、优化，直到通过或达到最大轮数。
- CI 门禁：聚合通过率 + 相对于基线的回归。

运行它：

```
python3 code/main.py
```

输出：每个用例的通过/失败、回归标记、CI 门禁裁决。

## 使用它

- 把评估用例写在与智能体代码相同的仓库中。
- 通过 CI 在每个 PR 上运行它们。
- 出现回归时让构建失败。
- 跟踪通过率随时间的变化。
- 把每一个生产失败都关联到一个新用例。

## 交付它

`outputs/skill-eval-suite.md` 为一个智能体产品构建一套带 CI 门禁和回归跟踪的三层评估套件。

## 练习

1. 选取你的一个生产失败。写一个能复现它的评估用例。你的智能体现在能通过吗？
2. 为你的领域构建一个 LLM 评审评分标准，包含三个维度（事实性、语气、范围）。为 50 个会话打分。
3. 把评估套件接入 CI。当回归 >=5% 时让构建失败。
4. 添加一个轨迹效率指标：相对于标准答案轨迹，智能体走了多少步？
5. 把 Phase 14 的每一节课都映射到你套件中的一个评估用例。有遗漏吗？那就是需要补上的缺口。

## 关键术语

| 术语 | 人们怎么说 | 它实际的含义 |
|------|----------------|------------------------|
| Static benchmark | "现成的评估" | SWE-bench、GAIA、AgentBench、WebArena、OSWorld |
| Custom offline eval | "领域评估" | 在你产品形态上的 LLM-as-judge / 执行 / 轨迹评估 |
| Online eval | "生产评估" | 会话回放、护栏告警、成本/延迟追踪 |
| Evaluator-optimizer | "提议-判断-优化" | 迭代直到评审通过 |
| CI gate | "合并阻断器" | 评估出现回归时让构建失败 |
| Baseline | "最近已知良好" | 用于检测回归的参考分数 |
| Trajectory efficiency | "相对标准答案的步数" | 智能体步数除以人类专家的最少步数 |

## 延伸阅读

- [Anthropic, Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) —— "从简单开始，用评估优化"
- [OpenAI, SWE-bench Verified](https://openai.com/index/introducing-swe-bench-verified/) —— 经过精选的基准
- [Berkeley Function Calling Leaderboard](https://gorilla.cs.berkeley.edu/leaderboard.html) —— 工具使用基准
- [Langfuse docs](https://langfuse.com/) —— 评估 + 会话回放的实践
