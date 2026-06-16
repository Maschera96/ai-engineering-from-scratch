# Benchmarks：SWE-bench，GAIA，AgentBench

> 2026 年有三个基准锚定了智能体评估。SWE-bench 测试代码补丁能力。GAIA 测试通用工具使用。AgentBench 测试多环境推理。要了解它们的构成、污染状况，以及它们没有测量什么。

**Type:** 学习
**Languages:** Python (stdlib)
**Prerequisites:** Phase 14 · 06 (Tool Use)
**Time:** ~60 minutes

## 学习ing Objectives

- 说出 SWE-bench 的测试框架（FAIL_TO_PASS），并解释它为何以单元测试为门槛。
- 解释 SWE-bench Verified（OpenAI，500 个任务）为何存在，以及它移除了什么。
- 描述 GAIA 的设计：对人类简单、对 AI 困难；三个难度等级。
- 说出 AgentBench 的八个环境，以及开源 LLM 的主要瓶颈。
- 概括 SWE-bench+ 的污染发现及其影响。

## 问题

排行榜告诉你哪个模型在某个基准上获胜。它们没有告诉你：

- 该基准是否被污染（训练数据中包含解答，测试泄漏）。
- 该基准是否测量你关心的内容（代码 vs 浏览 vs 通用）。
- 评估器是否稳健（AST 匹配、状态检查、人工审核）。

在引用某个数字之前，先了解这三个锚定基准及其失效模式。

## 概念

### SWE-bench (Jimenez et al.，ICLR 2024 oral)

- 来自 12 个流行 Python 仓库的 2,294 个真实 GitHub issue。
- 智能体得到：修复前 commit 状态的代码库 + 自然语言的 issue 描述。
- 智能体产出：一个补丁。
- 评估器：应用补丁，运行仓库的测试套件。补丁必须让 FAIL_TO_PASS 测试翻转（之前失败、现在通过），同时不破坏 PASS_TO_PASS 测试。

SWE-agent（Yang et al.，2024）在发布时通过强调智能体-计算机接口（文件编辑器命令、模型能理解的搜索语法）达到了 12.5%。

### SWE-bench Verified

OpenAI，2024 年 8 月。人工筛选的 500 个任务子集。移除了模糊的 issue、不可靠的测试，以及修复方案不明确的任务。回答"你的智能体能否交付真实补丁？"的主要基准。

### Contamination

- SWE-bench 中超过 94% 的 issue 早于大多数模型的知识截止时间。
- **SWE-bench+** 发现 32.67% 的成功补丁在 issue 文本中泄漏了解答（模型在描述中看到了修复方案），另有 31.08% 因测试覆盖不足而可疑。
- Verified 更干净，但并非完全没有污染。

实际影响：一个在 SWE-bench 上得分 50% 的模型，在 SWE-bench+ 上可能只得 35%。如果你声称 SWE-bench 性能，请始终同时报告两者。

### GAIA (Mialon et al.，Nov 2023)

- 466 个问题；其中 300 个保留用于 huggingface.co/gaia-benchmark 上的私有排行榜。
- 设计理念："对人类概念上简单（92%），但对 AI 困难（带插件的 GPT-4：15%）。"
- 测试推理、多模态、网络、工具使用。
- 三个难度等级；Level 3 需要跨多种模态的长工具链。

GAIA 是你用来衡量"通用能力"的基准。不要与代码专项基准混淆。

### AgentBench (Liu et al.，ICLR 2024)

- 8 个环境，涵盖代码（Bash、DB、KG）、游戏（Alfworld、LTP）、网络（WebShop、Mind2Web）以及开放式生成。
- 多轮，每个 分离 约 4k-13k 轮。
- 主要发现：长期推理、决策和指令遵循是开源 LLM 追赶商业模型的瓶颈。

### What these做not measure

- 真实世界的运营成本（token、墙钟时间）。
- 对抗条件下的安全行为。
- 在你的领域上的性能（使用你自己的评估，Lesson 30）。
- 尾部失败（基准取平均值；生产运营者关心最差的 1%）。

### Where benchmarking goes wrong

- **单一数字执念。** SWE-bench 50% 提供的信息少于 P50/P75/P95 成本 + 步数分布。
- **被污染的声明。** 报告 SWE-bench 却不提及 Verified 或 SWE-bench+ 是误导性的。
- **将基准作为开发目标。** 为基准优化会偏离生产可用性。

## 构建 It

`code/main.py` 实现了一个玩具级的类 SWE-bench 框架：

- 合成的 bug 修复任务（3 个任务）。
- 一个提出补丁的脚本化"智能体"。
- 一个检查 FAIL_TO_PASS（bug 现在已修复）和 PASS_TO_PASS（没有破坏任何东西）的测试运行器。
- 一个基于问题分解深度的 GAIA 风格难度分类器。

运行它：

```
python3 code/main.py
```

输出显示每个任务 + 每个难度的解决率，并使评估器规则具体化。

## Use It

- **SWE-bench Verified** 用于代码智能体。始终报告 Verified 分数。
- **GAIA** 用于通用智能体。使用私有排行榜 分离。
- **AgentBench** 用于多环境比较。
- **自定义评估**（Lesson 30）用于你产品的实际形态。

## Ship It

`outputs/skill-benchmark-harness.md` 为任意代码库-任务对构建一个 SWE-bench 风格的框架，带有 FAIL_TO_PASS / PASS_TO_PASS 门控。

## Exercises

1。将玩具框架移植到一个真实仓库上运行（选你自己的一个）。为已知 bug 编写 3 个 FAIL_TO_PASS 测试。
2。添加一个步数指标。在你的 3 个任务上，每次解决需要多少智能体步数？
3。阅读 SWE-bench+ 论文。实现一个解答泄漏检查（将 issue 文本与 diff 做模式匹配）。
4。从公开 分离 下载一个 GAIA 问题。追踪一个 GPT-4 级别的智能体会怎么做。它需要哪些工具？
5。阅读 AgentBench 的逐环境细分。哪个环境最贴近你的产品界面？那里的"SOTA"是什么样子？

## Key 术语s

| 术语 | What people say | What它actually 意味着什么 |
|------|----------------|------------------------|
| SWE-bench | "代码智能体基准" | 2,294 个 GitHub issue；补丁必须翻转 FAIL_TO_PASS 测试 |
| SWE-bench Verified | "干净的 SWE-bench" | 500 个人工筛选的任务，OpenAI |
| FAIL_TO_PASS | "修复门槛" | 之前失败、打补丁后必须通过的测试 |
| PASS_TO_PASS | "无回归门槛" | 之前通过、且必须仍然通过的测试 |
| GAIA | "通用基准" | 466 个人类易/AI 难的多工具问题 |
| AgentBench | "多环境基准" | 8 个环境；长时程多轮 |
| Contamination | "训练集泄漏" | 基准任务出现在模型训练中 |
| SWE-bench+ | "污染审计" | 在成功的 SWE-bench 补丁中发现 32.67% 的解答泄漏 |

## Further Reading

- [Jimenez et al.，SWE-bench (arXiv:2310.06770)](https://arxiv.org/abs/2310.06770) — 原始基准
- [OpenAI，SWE-bench Verified](https://openai.com/index/introducing-swe-bench-verified/) — 筛选后的子集
- [Mialon et al.，GAIA (arXiv:2311.12983)](https://arxiv.org/abs/2311.12983) — 通用基准
- [Liu et al.，AgentBench (arXiv:2308.03688)](https://arxiv.org/abs/2308.03688) — 多环境套件
