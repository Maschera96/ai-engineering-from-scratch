# Tree of Thoughts 与 LATS：深思熟虑的搜索

> 单条思维链轨迹没有回溯的余地。ToT（Yao 等人，2023）把推理变成一棵树，并在每个节点上进行自我评估。LATS（Zhou 等人，2024）在蒙特卡洛树搜索（MCTS）框架下将 ToT 与 ReAct、Reflexion 统一起来。Game of 24 从 4%（CoT）提升到 74%（ToT）；LATS 在 HumanEval 上达到 92.7% 的 pass@1。

**类型：** 构建
**语言：** Python（标准库）
**前置条件：** Phase 14 · 01（智能体循环）、Phase 14 · 03（Reflexion）
**时长：** ~75 分钟

## 学习目标

- 把推理视为搜索：节点是“思维”，边是“扩展”，价值是“有多大希望”。
- 用标准库实现一个 ToT 风格的 BFS 树搜索，并带有自我评估打分。
- 扩展成一个玩具版 LATS MCTS 循环，包含选择 / 扩展 / 模拟 / 反向传播。
- 判断什么时候搜索值得付出 token 倍增的代价（Game of 24、代码生成），什么时候单条轨迹就够了（简单问答）。

## 问题

思维链是一种线性行走。如果第一步错了，后续每一步都建立在错误的前提之上。在 Game of 24（用四个数字加上 + − × ÷ 凑出 24）上，GPT-4 的 CoT 准确率只有 4%。模型早早选错了子表达式，之后再也无法恢复。

推理真正需要的能力是：提出多个候选、评估它们、挑出有希望的、并在遇到死胡同时回溯。这就是搜索。Tree of Thoughts 和 LATS 是两种经典的表述方式。

## 概念

### Tree of Thoughts（Yao 等人，NeurIPS 2023）

每个节点都是一个连贯的中间步骤（“一个思维”）。每个节点可以扩展出 K 个子思维。LLM 用一个打分提示对每个节点进行自我评估。搜索在这棵树上进行探索——BFS、DFS 或束搜索。

```
                     (root："find 24 from 4 6 4 1")
                    /               |            \
           ("6 - 4 = 2")    ("4 + 1 = 5")    ("4 * 6 = 24")  <- 评分：HIGH
              /   \              |                  |
         。.。  。.。        。.。               finish
```

自我评估是承重的关键部件。论文展示了三种变体：`sure / likely / impossible` 分类、`1..10` 数值打分，以及在候选间投票。在 Game of 24 上，这三种方式都大幅超越 CoT（用 GPT-4 从 4% -> 74%）。

### LATS（Zhou 等人，ICML 2024）

LATS 在 MCTS 框架下统一了 ToT、ReAct 和 Reflexion。LLM 扮演三种角色：

- **策略（Policy）**：提出候选的下一步动作（ReAct 风格）。
- **价值函数（Value function）**：为一条部分轨迹打分（ToT 风格的自我评估）。
- **自我反思器（Self-reflector）**：在失败时写下一段自然语言反思（Reflexion 风格），并用它为未来的 rollout 重新播种。

环境反馈（观察）会混入价值函数，使搜索由真实的工具结果引导，而不仅仅是模型的意见。论文发表时的结果：用 GPT-4 在 HumanEval 上 pass@1 达到 92.7%（SOTA），用 GPT-3.5 在 WebShop 上平均得分 75.9（接近基于梯度的微调）。

### 极简版 MCTS

每次迭代包含四个阶段：

1。**选择（Select）**——使用 UCT（树的置信上界）从根走到一个叶子。
2。**扩展（Expand）**——通过策略生成 K 个子节点。
3。**模拟（Simulate）**——从某个子节点用策略进行 rollout，用价值函数（或环境奖励）为叶子打分。
4。**反向传播（Backpropagate）**——沿路径向上更新访问次数和价值估计。

UCT 公式：`Q(s，a) + c * sqrt(ln N(s) / N(s，a))`。第一项是利用（exploitation）；第二项是探索（exploration）。针对每个任务调节 `c`。

### 成本的现实

搜索会让 token 数量爆炸。ToT 在 Game of 24 上使用的 token是CoT 的 100–1000 倍。LATS 也类似。这不是免费的；把搜索保留给以下场景：

- 单条轨迹明显不够用的任务（Game of 24、复杂代码）。
- 相比正确性，墙钟时间没那么重要的任务。
- 拥有廉价、可靠的价值函数的任务（代码的单元测试、数学的明确目标）。

如果你的任务只有一个正确答案，而评估器又有噪声，那么搜索往往会让事情更糟——它会找到一个“分数很高”的错误答案。

### 2026 年的定位

大多数生产环境中的智能体并不运行 LATS。它们运行的是带有工具落地校验的 ReAct（CRITIC，第 05 课）。搜索出现在一些专门的细分领域：

- 把测试作为价值函数运行的编码智能体（HumanEval 风格）。
- 探索多条查询路径的深度研究智能体。
- LangGraph 子图内部以规划为主的工作流。

AlphaEvolve（第 11 课）是 2025 年的极端案例：对代码进行进化式搜索，使用机器可校验的适应度，取得前沿突破（56 年来首次改进 4x4 矩阵乘法）。

## 动手构建

`code/main.py` 实现了：

- 在一个风格化的“挑选算术运算”任务上的微型 ToT BFS。
- 在同一任务上的玩具版 LATS MCTS 循环（选择 / 扩展 / 模拟 / 反向传播），带 UCT 选择。
- 一个价值函数，由符号化分数加上自我评估分数组合而成。

运行它：

```
python3 code/main.py
```

这段轨迹展示了 ToT 用 BFS 在每个节点扩展三个候选，与 LATS 通过 MCTS 收敛到最佳 rollout 的对比。两者都会打印 token 计数。

## 使用它

LangGraph 以子图模式提供了 ToT 风格的探索；LangChain 团队关于 LATS 的博客（2024 年 5 月）是参考教程。LlamaIndex 提供了一个 `TreeOfThoughts` 智能体。对于大多数 2026 年的生产智能体，这种模式藏在一个 `if task_complexity > threshold：use_search()` 的门控之后——参见第 05 课中的 evaluator-optimizer 模式。

## 交付它

`outputs/skill-search-policy.md` 会根据任务形态、预算和评估器保真度，在线性 ReAct、ToT、LATS 和进化式搜索之间做选择。

## 练习

1。用 UCT c=0.1 与 c=2.0 分别运行玩具版 LATS。轨迹有什么变化？
2。把价值函数换成一个噪声更大的打分器（加入随机抖动）。MCTS 还能找到最佳叶子吗？它能容忍的最低信噪比是多少？
3。实现束搜索版 ToT（在每一层保留 top-k），并与 BFS 对比。在紧张的 token 预算下哪个更好？
4。阅读 LATS 第 5.1 节。复现 HumanEval 的轨迹数量：需要多少次 rollout 才能达到论文报告的 pass@1？
5。阅读 LATS 论文中关于“什么时候 LATS 帮助更小”的讨论。写一段话的决策规则，把任务形态映射到搜索策略。

## 关键术语

| 术语 | 人们怎么说 | 它实际的含义 |
|------|----------------|------------------------|
| Tree of Thoughts | “分支化的 CoT” | Yao 等人——带自我评估的思维节点树 |
| LATS | “面向 LLM 的 MCTS” | Zhou 等人——在 MCTS 下统一 ToT + ReAct + Reflexion |
| UCT | “置信上界” | 平衡利用（Q）与探索（ln N / n）的选择公式 |
| 价值函数 | “这个状态有多好” | 提示得到的 LLM 分数或环境奖励；喂给反向传播 |
| 策略 | “动作提议者” | ReAct 风格的生成器；产出候选的下一步思维/动作 |
| Rollout | “模拟的轨迹” | 用策略从一个节点走到叶子，再用价值函数打分 |
| 反向传播 | “更新祖先节点” | 把叶子的奖励沿路径向上推送，更新访问次数和 Q |
| 搜索成本 | “token 爆炸” | 在 Game of 24 上是 CoT 的 100-1000 倍；采用前先做预算 |

## 延伸阅读

- [Yao et al.，Tree of Thoughts (arXiv:2305.10601)](https://arxiv.org/abs/2305.10601) —— 经典论文
- [Zhou et al.，LATS (arXiv:2310.04406)](https://arxiv.org/abs/2310.04406) —— 带 Reflexion 反馈的 MCTS
- [LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview) —— 用于搜索的子图模式
- [AlphaEvolve (arXiv:2506.13131)](https://arxiv.org/abs/2506.13131) —— 带程序化评估器的进化式搜索
