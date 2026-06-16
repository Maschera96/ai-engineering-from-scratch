# MDP、状态、行动和奖励

> 马尔可夫决策过程有五件事：状态、行动、转换、奖励、折扣。 RL 中的所有内容（Q-learning、PPO、DPO、GRPO）都在此形状上进行优化。学习一次，免费阅读强化学习的其余部分。

**类型：** 学习
**语言：** Python
**先修：** 第 1 阶段·06（概率与分布），第 2 阶段·01（机器学习分类法）
**时间：** 约45分钟

## 问题

您正在编写一个国际象棋机器人。或者库存计划员。或者是贸易代理。或者训练推理模型的 PPO 循环。四个不同的领域，一个令人惊讶的事实：所有四个领域都崩溃为同一个数学对象。

监督学习为您提供 `(x, y)` 对，并要求您拟合一个函数。强化学习不会给你任何标签，只有状态流、你采取的行动和标量奖励。此举赢得了比赛吗？补货决定省钱了吗？交易有盈利吗？ LLM刚刚产出的词元是否给了法官更高的奖励？

在将其正式化之前，您无法从该流中学习。 “我看到了什么”、“我做了什么”、“接下来发生了什么”、“那有多好”，每一个都必须成为你可以推理的对象。这种形式化就是马尔可夫决策过程。此阶段的每个 RL 算法（包括最后的 RLHF 和 GRPO 循环）都会对此形状进行优化。

## 概念

![马尔可夫决策过程：状态、动作、转换、奖励、折扣](../assets/mdp.svg)

**五个物体。**

- **状态** `S`。一切都需要代理人来决定。在 GridWorld 中，单元格。在国际象棋中，棋盘。在法学硕士中，上下文窗口加上任何内存。
- **行动** `A`。选择。移动 up/down/left/right. 进行移动。发出一个词元。
- **过渡** `P(s' | s, a)`。给定状态 `s` 和动作 `a`，下一个状态的分布。国际象棋中的确定性，库存中的随机性，LLM 解码中的几乎确定性。
- **奖励** `R(s, a, s')`。标量信号。赢=+1，输=-1。收入减去成本。 GRPO 中的对数似然比项。
- **折扣** `γ ∈ [0, 1)`。未来的奖励与现在的奖励相比有多少。 `γ = 0.99` 购买约 100 步的地平线； `γ = 0.9` 购买 ~10 个。

**马尔可夫性质** `P(s_{t+1} | s_t, a_t) = P(s_{t+1} | s_0, a_0, …, s_t, a_t)`。未来只取决于现在的状态。如果不是，则状态表示是不完整的，不是方法的失败，而是状态的失败。

**策略和回报。** 策略 `π(a | s)` 将状态映射到操作分布。回报 `G_t = r_t + γ r_{t+1} + γ² r_{t+2} + …` 是未来奖励的贴现总和。值 `V^π(s) = E[G_t | s_t = s]` 是策略 `π` 下从 `s` 开始的预期回报。 Q 值 `Q^π(s, a) = E[G_t | s_t = s, a_t = a]` 是从特定操作开始的预期回报。每个 RL 算法都会估计这两者之一，然后相应地改进 `π`。

**贝尔曼方程。** 此阶段中的所有内容都使用的定点方程：

`V^π(s) = Σ_a π(a|s) Σ_{s', r} P(s', r | s, a) [r + γ V^π(s')]`
`Q^π(s, a) = Σ_{s', r} P(s', r | s, a) [r + γ Σ_{a'} π(a'|s') Q^π(s', a')]`

这些将预期回报分为“此步骤的奖励”加上“您着陆地点的折扣值”。递归。第 9 阶段中的每种算法要么迭代该方程以收敛（动态规划），从中采样（蒙特卡罗），要么引导它一步（时间差）。

```figure
discount-horizon
```

## 构建它

### 第 1 步：一个微小的确定性 MDP

4×4 网格世界。代理从左上角开始，右下角结束，每步奖励 -1，动作 `{up, down, left, right}`。参见 `code/main.py`。

```python
GRID = 4
TERMINAL = (3, 3)
ACTIONS = {"up": (-1, 0), "down": (1, 0), "left": (0, -1), "right": (0, 1)}

def step(state, action):
    if state == TERMINAL:
        return state, 0.0, True
    dr, dc = ACTIONS[action]
    r, c = state
    nr = min(max(r + dr, 0), GRID - 1)
    nc = min(max(c + dc, 0), GRID - 1)
    return (nr, nc), -1.0, (nr, nc) == TERMINAL
```

五行。这就是整个环境。确定性转换、恒定步长惩罚、吸收终端状态。

### 第二步：推出策略

策略是从状态到动作分布的函数。最简单：均匀随机。

```python
def uniform_policy(state):
    return {a: 0.25 for a in ACTIONS}

def rollout(policy, max_steps=200):
    s, total, steps = (0, 0), 0.0, 0
    for _ in range(max_steps):
        a = sample(policy(s))
        s, r, done = step(s, a)
        total += r
        steps += 1
        if done:
            break
    return total, steps
```

运行随机策略 1000 次。对于这个 4×4 板，平均回报约为 -60 到 -80。最佳回报是-6（右下直线路径）。缩小这一差距是第九阶段的一切。

### 步骤 3：通过贝尔曼方程精确计算 `V^π`

对于小型 MDP，贝尔曼方程是一个线性系统。枚举状态，应用期望，迭代直到值停止变化。

```python
def policy_evaluation(policy, gamma=0.99, tol=1e-6):
    V = {s: 0.0 for s in all_states()}
    while True:
        delta = 0.0
        for s in all_states():
            if s == TERMINAL:
                continue
            v = 0.0
            for a, pi_a in policy(s).items():
                s_next, r, _ = step(s, a)
                v += pi_a * (r + gamma * V[s_next])
            delta = max(delta, abs(v - V[s]))
            V[s] = v
        if delta < tol:
            return V
```

这是迭代的策略评估。它是 Sutton & Barto 的第一个算法，也是后续每个 RL 方法的理论基础。

### 步骤4：`γ`是一个有物理意义的超参数

有效范围大致为`1 / (1 - γ)`。 `γ = 0.9` → 10 个步骤。 `γ = 0.99` → 100 步。 `γ = 0.999` → 1000 步。

太低，代理的行为就会目光短浅。太高了，信用分配就会变得嘈杂，因为许多早期步骤都对遥远的未来奖励负有责任。 LLM RLHF 通常使用 `γ = 1`，因为情节短且有限制。控制任务使用`0.95–0.99`。长视野策略游戏使用`0.999`。

## 常见陷阱

- **非马尔可夫状态。** 如果您需要最后三个观察结果来决定，那么“状态”不仅仅是当前的观察结果。修复：堆栈帧（Atari 堆栈 4 上的 DQN）或使用循环状态（观测值上的 LSTM/GRU）。
- **稀疏奖励。** 仅限胜利的奖励使得在大型状态空间中学习几乎不可能。塑造奖励（中间信号）或模仿引导（阶段 9·09）。
- **奖励黑客行为。** 优化代理奖励通常会产生病态行为。 OpenAI 的赛艇特工一直在原地打转以收集能量，而不是完成比赛。始终根据目标结果而不是代理来定义奖励。
- **折扣错误规格。** `γ = 1` 在无限范围任务上使每个值都是无限的。始终以有限范围或 `γ < 1` 为上限。
- **奖励规模。** {+100, -100} 与 {+1, -1} 的奖励给出相同的最优策略，但梯度大小截然不同。在插入 PPO/DQN. 之前标准化为 `[-1, 1]`-ish

## 使用它

2026 堆栈在接触代码之前将每个 RL 管道简化为 MDP：

| 情况 | 状态 | 行动 | 报酬 | γ |
|-----------|-------|--------|--------|---|
| 控制（运动、操纵） | 关节角度 + 速度 | 连续扭矩 | 任务特定形状 | 0.99 |
| 游戏（国际象棋、围棋、扑克） | 董事会+历史 | 合法搬家 | 赢=+1 / 输=-1 | 1.0（有限） |
| 库存/定价 | 库存+需求 | 订单数量 | 收入-成本 | 0.95 |
| LLM 的 RLHF | 上下文标记 | 下一个词元 | 最后奖励模型得分 | 1.0（剧集约 200 个词元） |
| GRPO 推理 | 及时+部分响应 | 下一个词元 | 最后验证者 0/1 | 1.0 |

在编写任何训练循环之前编写五个元组。大多数“强化学习不起作用”的错误报告都可以追溯到纸面上被破坏的 MDP 公式。

## 交付它

另存为`outputs/skill-mdp-modeler.md`：

```markdown
---
name: mdp-modeler
description: Given a task description, produce a Markov Decision Process spec and flag formulation risks before training.
version: 1.0.0
phase: 9
lesson: 1
tags: [rl, mdp, modeling]
---

Given a task (control / game / recommendation / LLM fine-tuning), output:

1. State. Exact feature vector or tensor spec. Justify Markov property.
2. Action. Discrete set or continuous range. Dimensionality.
3. Transition. Deterministic, stochastic-with-known-model, or sample-only.
4. Reward. Function and source. Sparse vs shaped. Terminal vs per-step.
5. Discount. Value and horizon justification.

Refuse to ship any MDP where the state is non-Markovian without explicit mention of frame-stacking or recurrent state. Refuse any reward that was not defined in terms of the target outcome. Flag any `γ ≥ 1.0` on an infinite-horizon task. Flag any reward range >100x the typical step reward as a likely gradient-explosion source.
```

## 练习

1. **简单。** 在 `code/main.py` 中实现 4×4 GridWorld 和随机策略推出。播放 10,000 集。报告回报的平均值和标准差。与最佳回报 (-6) 进行比较。
2. **中。** 针对均匀随机策略运行 `policy_evaluation` 和 `γ ∈ {0.5, 0.9, 0.99}`。将 `V` 打印为 4×4 网格。解释为什么 `γ` 越大，终端附近的状态值增长得越快。
3. **硬。** 将 GridWorld 设为随机：每个动作以 `p = 0.1` 的概率滑向相邻方向。重新评估统一策略。 `V[start]` 变得更好还是更差？为什么？

## 关键术语

| 学期 | 人们怎么说 | 它实际上意味着什么 |
|------|-----------------|-----------------------|
| MDP | 《强化学习设置》 | 元组 `(S, A, P, R, γ)` 满足马尔可夫性质。 |
| 状态 | “代理人看到了什么” | 所选策略类别下未来动态的足够统计数据。 |
| 策略 | 「代理人的行为」 | 条件分布 `π(a \| s)` or deterministic map `s → a`。 |
| 返回 | 「总奖励」 | 当前步骤的折扣总和 `Σ γ^t r_t`。 |
| 价值 | “一个国家有多好” | 从 `s` 开始，`π` 下的预期回报。 |
| Q值 | “一个动作有多好” | `π` 下的预期回报从 `s` 开始，第一个行动为 `a`。 |
| 贝尔曼方程 | 《动态规划递归》 | 将价值/Q 定点分解为一步奖励加上折扣后继价值。 |
| 折扣`γ` | 《未来与现在》 | 远期奖励的几何权重；有效层位`~1/(1-γ)`。 |

## 延伸阅读

- [萨顿和巴托 (2018)。强化学习：简介，第二版](http://incompleteideas.net/book/RLbook2020.pdf) — 教科书。章。 3 涵盖 MDP 和贝尔曼方程；章。 1 激发了作为后续每一课基础的奖励假设。
- [贝尔曼（1957）。动态规划](https://press.princeton.edu/books/paperback/9780691146683/dynamic-programming) — 贝尔曼方程的起源。
- [OpenAI Spinning Up — 第 1 部分：关键概念](https://spinningup.openai.com/en/latest/spinningup/rl_intro.html) — 从深度 RL 角度简明 MDP 入门。
- [普特曼（2005）。马尔可夫决策过程](https://onlinelibrary.wiley.com/doi/book/10.1002/9780470316887) — MDP 和精确求解方法的运筹学参考。
- [利特曼（1996）。顺序决策算法（博士论文）](https://www.cs.rutgers.edu/~mlittman/papers/thesis-main.pdf) - MDP 作为动态规划专业的最清晰的推导。
