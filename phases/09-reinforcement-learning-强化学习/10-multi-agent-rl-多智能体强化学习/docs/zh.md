# Multi-Agent RL

> Single-agent RL assumes the 环境 is stationary. Put two 学习 agents in the same world and that assumption breaks: each 智能体 is part of the other's 环境, and both are changing. Multi-agent RL is the set of tricks to make 学习 converge when the Markov assumption no longer holds.

**类型：** Build
**语言：** Python
**先修：** Phase 9 · 04 (Q-learning), Phase 9 · 06 (REINFORCE), Phase 9 · 07 (Actor-Critic)
**时间：** 约 45 分钟

## 问题

一个robot 学习 to navigate a room is a single-agent RL problem. A soccer team is not. AlphaStar vs StarCraft opponents is not. A marketplace of bidding agents is not. Two cars negotiating a four-way stop is not. Many-on-many real-world problems are not.

In every multi-agent setting, from the perspective of any one 智能体, the other agents *are* part of the 环境. As they learn and change their behavior, the 环境 becomes non-stationary. The Markov property — "next 状态 depends only on current 状态 and my 动作" — gets violated because the next 状态 also depends on what the *other* agents chose, and their policies are moving 目标.

这breaks tabular convergence proofs (Q-learning's guarantee assumes a stationary 环境). It breaks naive deep RL too: agents chase each other in loops, never converge to a stable 策略. You need multi-agent-specific techniques: centralized 训练 / decentralized execution, counterfactual baselines, league play, self-play.

2026 applications: robot swarms, traffic 路由, autonomous vehicle fleets, market simulators, multi-agent LLM systems (Phase 16), and any game with more than one intelligent player.

## 概念

![Four MARL regimes: indep, centralized critic, self-play, league](../assets/marl.svg)

**Formalism: Markov Game.** A 泛化 of MDP: states `S`, a joint 动作 `a = (a_1, …, a_n)`, transition `P(s' | s, a)`, and per-agent rewards `R_i(s, a, s')`. Each 智能体 `i` maximizes its own return under its own 策略 `π_i`. If rewards are identical, it is **fully cooperative**. If zero-sum, it is **adversarial**. If mixed, it is **general-sum**.

**Core challenges:**

- **Non-stationarity.** `P(s' | s, a_i)` from 智能体 `i`'s view depends on `π_{-i}`, which is changing.
- **Credit assignment.** With a shared 奖励, which 智能体 caused it?
- **Exploration coordination.** Agents must explore complementary strategies, not redundantly explore the same 状态.
- **Scalability.** The joint 动作 space grows exponentially in `n`.
- **Partial observability.** Each 智能体 sees only its own observation; the global 状态 is 隐藏.

**Four dominant regimes:**

**1. Independent Q-learning / independent PPO (IQL, IPPO).** Each 智能体 learns its own Q or 策略, treating others as part of the 环境. Simple, sometimes it works (especially with experience replay acting as a smoothing agent-modeling trick). Theoretical convergence: none. In practice: fine for loosely-coupled tasks, bad for tightly-coupled ones.

**2. Centralized 训练, decentralized execution (CTDE).** Most common modern paradigm. Each 智能体 has its own *策略* `π_i` that conditions on local observation `o_i` — standard decentralized execution at deployment. During *训练*, a centralized critic `Q(s, a_1, …, a_n)` conditions on the full global 状态 and joint 动作. Examples:
- **MADDPG** (Lowe et al. 2017): DDPG with a centralized critic per 智能体.
- **COMA** (Foerster et al. 2017): counterfactual 基线 — ask "what would my 奖励 have been if I'd taken 动作 `a'` instead?" — isolates my contribution.
- **MAPPO** / **IPPO** with shared critic (Yu et al. 2022): PPO with a centralized value 函数. Dominant in 2026 for cooperative MARL.
- **QMIX** (Rashid et al. 2018): value decomposition — `Q_tot(s, a) = f(Q_1(s, a_1), …, Q_n(s, a_n))` with monotonic mixing.

**3. Self-play.** Two copies of the same 智能体 play each other. The opponent's 策略 *is* my 策略 from a past snapshot. AlphaGo / AlphaZero / MuZero. OpenAI Five. Works best for zero-sum games; the 训练 信号 is symmetric.

**4. League play.** An extension of self-play to general-sum / adversarial environments: keep a population of past and current policies, 样本 an opponent from the league, 训练 against them. Adds exploiters (specialize in beating the current best) and main exploiters (specialize in beating exploiters). AlphaStar (StarCraft II). Needed when the game admits "rock-paper-scissors" strategy cycles.

**Communication.** Allow agents to send learned 消息 `m_i` to each other. Works in cooperative settings. Foerster et al. (2016) showed that differentiable inter-agent communication can be 训练后的 end-to-end. Today's LLM-based multi-agent systems (Phase 16) essentially communicate in natural language.

## 动手构建

这lesson uses a 6×6 GridWorld with two cooperative agents. They start in opposite corners and must reach a shared goal. Shared 奖励: `-1` per 步骤 while either 智能体 is still moving, `+10` when both arrive. See `code/main.py`.

### 步骤 1: the multi-agent env

```python
class CoopGridWorld:
    def __init__(self):
        self.size = 6
        self.goal = (5, 5)

    def reset(self):
        return ((0, 0), (5, 0))  # two agents

    def step(self, state, actions):
        a1, a2 = state
        new1 = move(a1, actions[0])
        new2 = move(a2, actions[1])
        done = (new1 == self.goal) and (new2 == self.goal)
        reward = 10.0 if done else -1.0
        return (new1, new2), reward, done
```

这个*joint* 动作 space is `|A|² = 16`. The global 状态 is two positions.

### 步骤 2: independent Q-learning

Each 智能体 runs its own Q-table keyed on joint 状态. At each 步骤: both pick ε-greedy actions, collect joint transition, each updates its own Q with the shared 奖励.

```python
def independent_q(env, episodes, alpha, gamma, epsilon):
    Q1, Q2 = defaultdict(default_q), defaultdict(default_q)
    for _ in range(episodes):
        s = env.reset()
        while not done:
            a1 = epsilon_greedy(Q1, s, epsilon)
            a2 = epsilon_greedy(Q2, s, epsilon)
            s_next, r, done = env.step(s, (a1, a2))
            target1 = r + gamma * max(Q1[s_next].values())
            target2 = r + gamma * max(Q2[s_next].values())
            Q1[s][a1] += alpha * (target1 - Q1[s][a1])
            Q2[s][a2] += alpha * (target2 - Q2[s][a2])
            s = s_next
```

Works on this 任务 because rewards are 稠密 and aligned. Fails on tightly-coupled tasks (e.g., where one 智能体 has to *wait* for the other).

### 步骤 3: centralized Q with decomposed-value update

使用one Q over joint actions `Q(s, a_1, a_2)`. Update from shared 奖励. Decentralize at execution by marginalizing: `π_i(s) = argmax_{a_i} max_{a_{-i}} Q(s, a_1, a_2)`. Trades exponential joint 动作 space for a *correct* global view.

### 步骤 4: simple self-play (adversarial 2-智能体)

Same 智能体, two roles. 训练 智能体 A against 智能体 B; after `K` episodes, copy A's 权重 into B. Symmetric 训练, consistent progress. The AlphaZero recipe in miniature.

## Pitfalls

- **Non-stationary replay.** Experience replay with independent agents is worse than single-agent because old transitions were 生成的 by now-obsolete opponents. 修复: re标签 or 权重 by recency.
- **Credit assignment ambiguity.** Shared 奖励 after a long episode; no clear way to say which 智能体 contributed. 修复: counterfactual baselines (COMA), or 奖励 shaping per 智能体.
- **策略 drift / chasing.** Each 智能体's best 响应 changes with each other's update. 修复: centralized critic, slow 学习 rates, or freeze-one-at-a-time.
- **奖励 hacking via coordination.** Agents find coordinated exploits the designer did not anticipate. Auction agents converge to bid zero. 修复: careful 奖励 design, behavioral constraints.
- **Exploration redundancy.** Both agents explore the same state-action pairs. 修复: 熵 bonuses per-agent, or role-conditioning.
- **League cycles.** Pure self-play can get stuck in a dominance cycle. 修复: league play with diverse opponents.
- **样本 explosion.** `n` agents × 状态 space × joint actions. Approximate with 函数 approximation; factored 动作 spaces (one 策略 输出 头 per 智能体).

## 实际使用

这个2026 MARL 应用 map:

|领域|Method|Notes|
|--------|--------|-------|
|Cooperative navigation / manipulation|MAPPO / QMIX|CTDE; shared critic + decentralized actors.|
|Two-player games (chess, Go, poker)|Self-play with MCTS (AlphaZero)|Zero-sum; symmetric 训练.|
|Complex multiplayer (Dota, StarCraft)|League play + imitation pretraining|OpenAI Five, AlphaStar.|
|Autonomous-vehicle fleets|CTDE MAPPO / PPO with 注意力|Partial obs; variable team sizes.|
|Auction markets|Game-theoretic equilibrium + RL|Mean-field RL when `n` → ∞.|
|LLM multi-agent systems (Phase 16)|Natural-language comm + role 条件控制|RL 循环 at the agent-planning 层.|

In 2026, MARL's biggest growth area is LLM-based: swarms of language-model agents negotiating, debating, building software. The RL shows up as preference 优化 on *trajectory-level* outputs, not token-level (Phase 16 · 03).

## 交付成果

Save as `outputs/skill-marl-architect.md`:

```markdown
---
name: marl-architect
description: Pick the right multi-agent RL regime (IPPO, CTDE, self-play, league) for a given task.
version: 1.0.0
phase: 9
lesson: 10
tags: [rl, multi-agent, marl, self-play]
---

Given a task with `n` agents, output:

1. Regime classification. Cooperative / adversarial / general-sum. Justify.
2. Algorithm. IPPO / MAPPO / QMIX / self-play / league. Reason tied to coupling tightness and reward structure.
3. Information access. Centralized training (what global info goes to the critic)? Decentralized execution?
4. Credit assignment. Counterfactual baseline, value decomposition, or reward shaping.
5. Exploration plan. Per-agent entropy, population-based training, or league.

Refuse independent Q-learning on tightly-coupled cooperative tasks. Refuse to recommend self-play for general-sum with cycle risks. Flag any MARL pipeline without a fixed-opponent eval (cherry-picked self-play numbers are common).
```

## 练习

1. **Easy.** 训练 independent Q-learning on the 2-智能体 cooperative GridWorld. How many episodes until mean return > 0? Plot the joint 学习 曲线.
2. **Medium.** Add a "coordination" 任务: the goal is reached only when both agents 步骤 onto it on the same turn. Does independent Q still converge? What breaks?
3. **Hard.** Implement a centralized critic for MAPPO-style 训练 and compare convergence speed to independent PPO on the coordination 任务.

## Key Terms

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|Markov game|"Multi-agent MDP"|`(S, A_1, …, A_n, P, R_1, …, R_n)`; each 智能体 has its own 奖励.|
|CTDE|"Centralized 训练, decentralized execution"|Joint critic at 训练 time; each 智能体's 策略 uses only local obs.|
|IPPO|"Independent PPO"|Each 智能体 runs PPO separately. Simple 基线; often underrated.|
|MAPPO|"Multi-agent PPO"|PPO with a centralized value 函数 conditioned on global 状态.|
|QMIX|"Monotonic value decomposition"|`Q_tot = f_monotone(Q_1, …, Q_n)` allows decentralized argmax.|
|COMA|"Counterfactual multi-agent"|Advantage = my Q minus expected Q marginalizing over my 动作.|
|Self-play|"智能体 vs past self"|Single 智能体, two roles; standard for zero-sum games.|
|League play|"Population 训练"|缓存 past policies, 样本 opponents from the pool; handles strategy cycles.|

## 延伸阅读

- [Lowe et al. (2017). Multi-Agent Actor-Critic for Mixed Cooperative-Competitive Environments (MADDPG)](https://arxiv.org/abs/1706.02275) — CTDE with a centralized critic.
- [Foerster et al. (2017). Counterfactual Multi-Agent Policy Gradients (COMA)](https://arxiv.org/abs/1705.08926) — counterfactual baselines for credit assignment.
- [Rashid et al. (2018). QMIX: Monotonic Value Function Factorisation](https://arxiv.org/abs/1803.11485) — value decomposition with monotonicity.
- [Yu et al. (2022). The Surprising Effectiveness of PPO in Cooperative Multi-Agent Games (MAPPO)](https://arxiv.org/abs/2103.01955) — PPO is surprisingly strong for MARL.
- [Vinyals et al. (2019). Grandmaster level in StarCraft II using multi-agent reinforcement learning (AlphaStar)](https://www.nature.com/articles/s41586-019-1724-z) — league play at 规模.
- [Silver et al. (2017). Mastering the game of Go without human knowledge (AlphaGo Zero)](https://www.nature.com/articles/nature24270) — pure self-play in zero-sum games.
- [Sutton & Barto (2018). Ch. 15 — Neuroscience & Ch. 17 — Frontiers](http://incompleteideas.net/book/RLbook2020.pdf) — includes the textbook's short treatment of multi-agent settings and the non-stationarity problem that CTDE is designed to solve.
- [Zhang, Yang & Başar (2021). Multi-Agent Reinforcement Learning: A Selective Overview](https://arxiv.org/abs/1911.10635) — survey covering cooperative, competitive, and mixed MARL with convergence results.
