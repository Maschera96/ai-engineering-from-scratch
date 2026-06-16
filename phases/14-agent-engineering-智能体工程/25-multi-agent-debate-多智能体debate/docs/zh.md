# 多智能体 Debate 与协作

> Du 等人（ICML 2024，"Society of Minds"）运行 N 个模型实例，让它们独立提出答案，然后在 R 轮中迭代地互相批评以达成收敛。提升了事实性、规则遵循能力和推理能力。稀疏拓扑在 token 成本上优于全连接网格。

**类型：** 学习 + 构建
**语言：** Python (stdlib)
**前置要求：** Phase 14 · 12（工作流模式）、Phase 14 · 05（Self-Refine 与 CRITIC）
**时长：** ~60 分钟

## 学习目标

- 解释 debate 协议：N 个提议者、R 轮、收敛到一个共享答案。
- 描述为什么 debate 能提升事实性、规则遵循能力和推理能力。
- 解释稀疏拓扑：并非每个辩论者都需要看到其他所有人。
- 用脚本化 LLM 通过 stdlib 实现 debate，包含全连接网格和稀疏两种变体；衡量 token 成本与准确率的关系。

## 问题所在

Self-Refine（第 05 课）是一个模型在批评自己——存在群体思维的风险。CRITIC（第 05 课）将批评建立在外部工具之上——但工具并不总是可用。Debate 引入了第三种模式：多个实例、交叉批评、通过分歧达成收敛。

## 概念

### Society of Minds（Du 等人，ICML 2024）

- N 个模型实例针对同一个问题独立提出答案。
- 在 R 轮中，每个模型阅读其他模型的提议并对其进行批评。
- 模型根据这些批评更新自己的答案。
- 经过 R 轮后，返回收敛后的答案。

最初的实验由于成本原因使用了 N=3、R=2。在困难问题（MMLU、GSM8K、Chess Move Validity、传记生成）上，更多的智能体和更多的轮次能提升准确率。

跨模型组合优于单模型 debate：ChatGPT + Bard 一起 > 任一单独使用。

### 稀疏拓扑

"Improving Multi-Agent Debate with Sparse Communication Topology"（arXiv:2406.11776，2024-2025）表明全连接网格 debate 并不总是最优的。稀疏拓扑（星型、环型、轮辐型）可以在更低的 token 成本下达到相同的准确率。每个辩论者只看到一部分同伴。

含义：

- 全连接网格 N=5、R=3 = 5 × 3 = 15 个提议，每个阅读 4 个同伴 = 60 次批评操作。
- 星型 N=5、R=3（一个中心 + 4 个辐条）= 15 个提议，辐条只阅读中心 = 12 次批评操作。

### debate 何时有帮助

- **事实性。** N 个独立提议，交叉核对可减少幻觉。
- **规则遵循。** Chess move validity——一个模型漏掉某条规则，其他模型能发现。
- **开放式推理。** 多种框定方式逐渐收敛到正确答案。

### debate 何时有害

- **延迟敏感的 UX。** N × R 串行轮次带来的延迟可能是你承受不起的。
- **成本敏感的规模化。** 每个问题需要 N × R 的 token。
- **简单的事实查找。** 一次查找比五次 debate 更便宜。

### 2026 年的实际落地形态

- **Anthropic 编排器-工作者**（第 12 课）——debate 的一种变体，带有综合步骤。
- **LangGraph supervisor**（第 13 课）——中心路由器 + 专家智能体，可以把 debate 实现为一个节点。
- **OpenAI Agents SDK**（第 16 课）——智能体之间来回交接以进行迭代批评。
- **多智能体评估**——将 debate + evaluator-optimizer 配对，用于评估信号。

### 这个模式会在哪里出错

- **收敛坍塌。** 所有智能体都收敛到第一个错误答案上。用强制分歧轮次来缓解。
- **中心失效。** 在星型拓扑中，一个糟糕的中心会污染所有人。轮换中心或使用多个中心。
- **提示同质化。** 所有智能体使用相同的提示；它们产生相同的答案。使用多样化的提示和/或模型。

## 动手构建

`code/main.py` 实现了 stdlib debate：

- `Debater` 类（带有每个辩论者观点漂移的脚本化 LLM）。
- `FullMeshDebate` 和 `SparseDebate` 运行器。
- 三个问题：一个事实性问题、一个基于规则的问题、一个推理问题。
- 指标：收敛答案、达到收敛的轮数、批评操作总数。

运行它：

```
python3 code/main.py
```

输出：每种协议的准确率和成本；稀疏在 3 个问题中的 2 个上以更低的成本达到与全连接网格相同的效果。

## 使用它

- **Anthropic 编排器-工作者**用于简单的 2-3 个工作者的 debate。
- **LangGraph** 用于带检查点的有状态多轮 debate。
- **自定义**用于研究或专门的正确性保证。

## 交付它

`outputs/skill-debate.md` 搭建了一个多智能体 debate 的脚手架，可配置拓扑、N、R 以及收敛规则。

## 练习

1. 实现一条"强制分歧"规则：在第 1 轮，每个辩论者都必须产生一个不同的提议。衡量它对收敛速度的影响。
2. 添加置信度加权聚合：辩论者返回 (answer, confidence)；聚合器按置信度加权。这有帮助吗？
3. 把一个"智能体"换成另一个观点不同的脚本化 LLM。异质性能提升准确率吗？
4. 在你的 3 个问题上衡量全连接网格与稀疏的 token 成本。绘制成本与准确率的关系图。
5. 阅读 Society of Minds 论文。把你的玩具示例移植到 N=5、R=3。什么会崩溃？什么会变得更好？

## 关键术语

| 术语 | 人们怎么说 | 它实际的含义 |
|------|----------------|------------------------|
| Debate | "多智能体批评" | N 个提议者、R 轮交叉批评、收敛 |
| Full mesh | "人人阅读人人" | 每个辩论者每轮都阅读每个同伴 |
| Sparse topology | "受限的同伴视图" | 辩论者只阅读一部分同伴 |
| Hub-and-spoke | "星型拓扑" | 一个中心辩论者，N-1 个辐条只阅读中心 |
| Convergence | "达成一致" | 辩论者收敛到一个共享答案 |
| Society of Minds | "Du 等人的 debate 论文" | ICML 2024 多智能体 debate 方法 |

## 延伸阅读

- [Du et al., Society of Minds (arXiv:2305.14325)](https://arxiv.org/abs/2305.14325) — 经典的多智能体 debate
- [Sparse Communication Topology (arXiv:2406.11776)](https://arxiv.org/abs/2406.11776) — 稀疏拓扑的结果
- [Anthropic, Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) — 作为 debate 变体的编排器-工作者
- [Madaan et al., Self-Refine (arXiv:2303.17651)](https://arxiv.org/abs/2303.17651) — 单模型自我批评的对应方法
