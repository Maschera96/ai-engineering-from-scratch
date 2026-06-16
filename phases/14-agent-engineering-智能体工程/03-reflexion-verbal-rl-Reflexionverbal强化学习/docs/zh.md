# Reflexion：Verbal 强化学习

> 基于梯度的 RL 需要数千次试验和一个 GPU 集群才能修复一种失败模式。Reflexion（Shinn et al.，NeurIPS 2023）用自然语言做到了这一点：每次试验失败后，智能体写下一段反思，将其存入情景记忆，并让下一次试验以该记忆为条件。这正是 Lett的 sleep-time compute、Claude Code 的 CLAUDE.md learnings 以及 pro-workflow 的 learn-rule 背后的模式。

**类型：** 构建
**语言：** Python (stdlib)
**前置要求：** Phase 14 · 01（Agent Loop）、Phase 14 · 02（ReWOO）
**时长：** ~60 分钟

## 学习目标

- 说出 Reflexion 的三个组件（Actor、Evaluator、Self-Reflector）以及情景记忆的作用。
- 用 stdlib 实现一个带二元评估器、反思缓冲区和全新重试的 Reflexion 循环。
- 针对给定任务在标量、启发式和自评估反馈来源之间做出选择。
- 解释为什么 verbal 强化能捕捉到那些基于梯度的 RL 需要数千次试验才能修复的错误。

## 问题

一个智能体在某个任务上失败了。在标准 RL 中，你会再运行数千次试验，计算梯度，更新权重。这既昂贵又缓慢，而且大多数生产环境的智能体并没有为每一次失败都准备训练预算。

Reflexion（Shinn et al.，arXiv:2303.11366）提出了一个不同的问题：如果智能体只是思考一下自己为什么失败，然后把这个想法放进提示里再试一次会怎样？没有权重更新。没有梯度。只有在试验之间存储的自然语言。

结果是：在 ALFWorld 上，它击败了 ReAct 和其他未经微调的基线。在 HotpotQA 上，它优于 ReAct。在代码生成（HumanEval/MBPP）上，它创下了当时的最高水平。所有这些都没有用到任何一步梯度。

## 概念

### 三个组件

```
Actor         ：generates trajectory (ReAct-style loop)
Evaluator     ：scorestrajectory — binary，heuristic，or self-eval
Self-Reflector：writes natural-language reflection onfailure
```

外加一个数据结构：

```
Episodic memory：list of prior reflections，prepended tonext trial's prompt
```

一次试验运行 Actor。Evaluator 给它打分。如果分数低，Self-Reflector 就产出一段反思（"我选错了工具，因为我把问题误读成在问 X，而它其实在问 Y"）。这段反思进入情景记忆。下一次试验从头开始，但能看到这段反思。

### 三种评估器类型

1。**标量（Scalar）** —— 一个外部的二元信号。ALFWorld 成功或失败。HumanEval 测试通过或失败。最简单，信号最强。
2。**启发式（Heuristic）** —— 预定义的失败特征。"如果智能体连续两次产出相同的动作，标记为卡住。""如果轨迹超过 50 步，标记为低效。"
3。**自评估（Self-evaluated）** —— LLM 给自己的轨迹打分。在没有真值可用时需要它。信号较弱；与工具接地的验证搭配使用效果好（Lesson 05 —— CRITIC）。

2026 年的默认做法是一种混合：有真值时用标量，没有时用自评估，启发式作为安全护栏。

### 为什么它能泛化

Reflexion 与其说是一个新算法，不如说是一个被命名的模式。几乎每一个生产环境中的"自愈"智能体都运行着它的某个变体：

- Lett的 sleep-time compute（Lesson 08）：一个独立的智能体反思过往对话并写入记忆块。
- Claude Code 的 `CLAUDE.md` / "save memory" 模式：将反思捕获为 learnings，添加到未来会话的开头。
- pro-workflow 的 `/learn-rule` 命令：将更正捕获为显式规则。
- LangGraph 的反思节点：一个给输出打分、并在需要时路由到精炼步骤的节点。

它们都源自同一个洞见：自然语言是一种足够丰富的媒介，能够在不同运行之间传递"我从失败中学到了什么"。

### 何时有效、何时无效

Reflexion 在以下情况有效：

- 存在清晰的失败信号（测试失败、工具报错、答案错误）。
- 任务类别可复现（同一类型的问题可以再次被提出）。
- 反思有改进轨迹的空间（有足够的动作预算）。

Reflexion 在以下情况帮不上忙：

- 智能体在第一次尝试时就已经成功。
- 失败是外部原因造成的（网络断了、工具坏了）—— 对"网络断了"做反思对未来的运行没有帮助。
- 反思变成了迷信 —— 为一次偶发的不稳定运行存储了一段叙述。

2026 年的陷阱：记忆腐烂（memory rot）。反思不断累积；其中一些已过时或错误；随着情景缓冲区增长，重新运行会变得越来越慢。缓解方法：周期性压缩（Lesson 06）、给反思设置 TTL，或者用一个独立的 sleep-time 清理智能体（Letta）。

```figure
react-trace
```

## 动手构建

`code/main.py` 在一个玩具谜题上实现了 Reflexion：产出一个总和等于目标值的 3 元素列表。Actor 发出候选列表；Evaluator 检查总和；Self-Reflector 写下一行说明哪里出了问题。这段反思进入情景记忆，供下一次试验使用。

组件：

- `Actor` —— 一个脚本化的策略，在看到反思时会改进。
- `Evaluator.binary()` —— 对目标总和做通过/失败判断。
- `SelfReflector` —— 为失败生成一行诊断。
- `EpisodicMemory` —— 一个带 TTL 语义的有界列表。

运行它：

```
python3 code/main.py
```

这段轨迹展示了三次试验。试验 1 失败，存储一段反思，试验 2 看到反思后有所改进但仍然失败，试验 3 成功。与一次基线运行（无反思）对比 —— 它会一直卡在试验 1 的答案上。

## 使用它

LangGraph 将反思作为一种节点模式提供。Claude Code 的 `/memory` 命令和 pro-workflow 的 `/learn-rule` 把情景缓冲区外化为一个 markdown 文件。Lett的 sleep-time compute 在停机期间运行 Self-Reflector，让主智能体保持受延迟约束。OpenAI Agents SDK 没有直接提供 Reflexion；你可以用一个按分数拒绝轨迹的自定义 Guardrail 和一个跨运行存活的记忆 `Session` 来构建它。

## 交付它

`outputs/skill-reflexion-buffer.md` 创建并维护一个情景缓冲区，具备反思捕获、TTL 和去重功能。给定一个任务类别和一次失败，它会产出一段真正能帮助下一次试验的反思（而不是泛泛的"更小心一点"）。

## 练习

1。从二元评估器切换到返回距离度量（离目标有多远）的标量评估器。它收敛得更快吗？
2。给反思添加 10 次试验的 TTL。在那之后，更旧的反思是有害还是有帮助？
3。实现启发式评估器：如果相同的动作重复，则把该试验标记为卡住。这与 Self-Reflector 如何相互作用？
4。用一个忽略反思的对抗性 Actor 来运行 Reflexion。要迫使 Actor 注意到反思，最少需要怎样的反思提示工程？
5。阅读 Reflexion 论文关于 AlfWorld 的第 4 节。在概念上复现那 130% 的成功率提升：相对于原版 ReAct，关键差异是什么？

## 关键术语

| 术语 | 人们怎么说 | 它实际的含义 |
|------|----------------|------------------------|
| Reflexion | "自我纠正" | Shinn et al。2023 —— Actor、Evaluator、Self-Reflector 外加情景记忆 |
| Verbal 强化 | "无梯度学习" | 添加到下一次试验提示开头的自然语言反思 |
| 情景记忆 | "按任务的反思" | 针对一个任务类别的、有界的先前反思缓冲区 |
| 标量评估器 | "二元成功信号" | 来自真值的通过/失败或数值分数 |
| 启发式评估器 | "基于模式的检测器" | 预定义的失败特征（例如卡住循环、步数过多） |
| 自评估器 | "对自己轨迹做 LLM-as-judge" | 没有真值时的低信号回退 —— 与工具接地的验证搭配使用 |
| 记忆腐烂 | "陈旧的反思" | 情景缓冲区被过时条目填满；用压缩/TTL 修复 |
| Sleep-time 反思 | "异步自我反思" | 在热路径之外运行 Self-Reflector，让主智能体保持快速 |

## 延伸阅读

- [Shinn et al.，Reflexion：Language Agents with Verbal Reinforcement 学习ing (arXiv:2303.11366)](https://arxiv.org/abs/2303.11366) —— 经典论文
- [Letta，Sleep-time Compute](https://www.letta.com/blog/sleep-time-compute) —— 生产环境中的异步反思
- [Anthropic，Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) —— 将情景缓冲区作为上下文的一部分进行管理
- [LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview) —— 反思节点模式
