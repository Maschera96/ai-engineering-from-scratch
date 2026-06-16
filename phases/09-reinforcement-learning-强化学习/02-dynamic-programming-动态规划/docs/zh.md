# Dynamic Programming — 策略 Iteration & Value Iteration

> Dynamic programming is RL with cheating. You already know the transition and 奖励 函数; you just iterate the Bellman equation until `V` or `π` stops moving. It is the 基准 every sampling-based method tries to approach.

**类型：** Build
**语言：** Python
**先修：** Phase 9 · 01 (MDPs)
**时间：** 约 75 分钟

## 问题

你have an MDP with a known 模型: you can 查询 `P(s' | s, a)` and `R(s, a, s')` for any state-action pair. An inventory manager knows the demand 分布. A board game has deterministic transitions. A gridworld is four lines of Python. You have a *模型*.

Model-free RL (Q-learning, PPO, REINFORCE) was invented for the case where you don't have a 模型 — you can only 样本 from the 环境. But when you do have one, there are faster, better methods: dynamic programming. Bellman designed them in 1957. They still define correctness: when people say "optimal 策略 for this MDP," they mean the 策略 DP would return.

你need them in 2026 for three reasons. First, every tabular 环境 in RL research (GridWorld, FrozenLake, CliffWalking) is solved with DP to produce the gold-standard 策略. Second, exact values let you *调试* 采样 methods: if Q-learning's estimate for `V*(s_0)` disagrees with the DP 答案 by 30%, your Q-learning has a bug. Third, modern offline RL and planning methods (MCTS, AlphaZero's search, model-based RL in Phase 9 · 10) all iterate a Bellman backup over a learned or given 模型.

## 概念

![Policy iteration and value iteration, side by side](../assets/dp.svg)

**Two algorithms, both fixed-point iteration on Bellman.**

**策略 iteration.** Alternates two 步骤 until the 策略 stops changing.

1. *评估:* given 策略 `π`, 计算 `V^π` by repeatedly applying `V(s) ← Σ_a π(a|s) Σ_{s',r} P(s',r|s,a) [r + γ V(s')]` until it converges.
2. *Improvement:* given `V^π`, make `π` greedy w.r.t. `V^π`: `π(s) ← argmax_a Σ_{s',r} P(s',r|s,a) [r + γ V(s')]`.

Convergence is guaranteed because (a) each improvement 步骤 either keeps `π` the same or strictly increases `V^π` for some 状态, (b) the space of deterministic policies is finite. Usually converges in ~5–20 outer iterations even for large 状态 spaces.

**Value iteration.** Collapses 评估 and improvement into one sweep. Apply the Bellman *optimality* equation:

`V(s) ← max_a Σ_{s',r} P(s',r|s,a) [r + γ V(s')]`

Repeat until `max_s |V_{new}(s) - V(s)| < ε`. Extract the 策略 at the end by taking the greedy 动作. Strictly faster per iteration — no inner 评估 循环 — but typically needs more iterations to converge.

**Generalized 策略 iteration (GPI).** The unifying framing. Value 函数 and 策略 are locked in a two-way improvement 循环; any method that drives both toward mutual consistency (异步 value iteration, modified 策略 iteration, Q-learning, actor-critic, PPO) is an instance of GPI.

**Why `γ < 1` matters.** The Bellman operator is a `γ`-contraction in the sup-norm: `||T V - T V'||_∞ ≤ γ ||V - V'||_∞`. Contraction implies unique fixed point and geometric convergence. Drop `γ < 1` and you lose the guarantee — you need a finite horizon or an absorbing terminal 状态.

```figure
value-iteration-gamma
```

## 动手构建

### 步骤 1: build the GridWorld MDP 模型

使用the same 4×4 GridWorld from Lesson 01. We add a stochastic variant: with 概率 `0.1` the 智能体 slips to a random perpendicular direction.

```python
SLIP = 0.1

def transitions(state, action):
    if state == TERMINAL:
        return [(state, 0.0, 1.0)]
    outcomes = []
    for direction, prob in action_probs(action):
        outcomes.append((apply_move(state, direction), -1.0, prob))
    return outcomes
```

`transitions(s, a)` returns a list of `(s', r, p)`. This is the entire 模型.

### 步骤 2: 策略 评估

给定a 策略 `π(s) = {action: prob}`, iterate the Bellman equation until `V` stops moving:

```python
def policy_evaluation(policy, gamma=0.99, tol=1e-6):
    V = {s: 0.0 for s in states()}
    while True:
        delta = 0.0
        for s in states():
            v = sum(pi_a * sum(p * (r + gamma * V[s_prime])
                              for s_prime, r, p in transitions(s, a))
                   for a, pi_a in policy(s).items())
            delta = max(delta, abs(v - V[s]))
            V[s] = v
        if delta < tol:
            return V
```

### 步骤 3: 策略 improvement

Replace `π` with the greedy 策略 w.r.t. `V`. If `π` did not change, return — we are at the optimum.

```python
def policy_improvement(V, gamma=0.99):
    new_policy = {}
    for s in states():
        best_a = max(
            ACTIONS,
            key=lambda a: sum(p * (r + gamma * V[s_prime])
                              for s_prime, r, p in transitions(s, a)),
        )
        new_policy[s] = best_a
    return new_policy
```

### 步骤 4: stitch them together

```python
def policy_iteration(gamma=0.99):
    policy = {s: "up" for s in states()}   # arbitrary start
    for _ in range(100):
        V = policy_evaluation(lambda s: {policy[s]: 1.0}, gamma)
        new_policy = policy_improvement(V, gamma)
        if new_policy == policy:
            return V, policy
        policy = new_policy
```

Typical convergence on 4×4: 4–6 outer iterations. Outputs `V*(0,0) ≈ -6` and a 策略 that strictly decreases the 步骤 count.

### 步骤 5: value iteration (the one-loop version)

```python
def value_iteration(gamma=0.99, tol=1e-6):
    V = {s: 0.0 for s in states()}
    while True:
        delta = 0.0
        for s in states():
            v = max(sum(p * (r + gamma * V[s_prime])
                       for s_prime, r, p in transitions(s, a))
                   for a in ACTIONS)
            delta = max(delta, abs(v - V[s]))
            V[s] = v
        if delta < tol:
            break
    policy = policy_improvement(V, gamma)
    return V, policy
```

Same fixed point, fewer lines of code.

## Pitfalls

- **Forgetting to handle terminals.** If you apply Bellman to an absorbing 状态, it still picks up a "best 动作" that changes nothing. Guard with `if s == terminal: V[s] = 0`.
- **Sup-norm vs L2 convergence.** Use `max |V_new - V|`, not average. The theoretical guarantee is on the sup-norm.
- **In-place vs synchronous updates.** Updating `V[s]` in-place (Gauss-Seidel) converges faster than a separate `V_new` dict (Jacobi). 生产 code uses in-place.
- **策略 ties.** If two actions have equal Q-value, `argmax` may break ties differently each iteration, causing the "策略 stable" check to oscillate. Use a stable tie-break (first 动作 in fixed order).
- **State-space explosion.** DP is `O(|S| · |A|)` per sweep. Works up to ~10⁷ states. Beyond that, you need 函数 approximation (Phase 9 · 05 onwards).

## 实际使用

In 2026, DP is the correctness 基线 and the inner 循环 of planners:

|Use case|Method|
|----------|--------|
|Solve a small tabular MDP exactly|Value iteration (simpler) or 策略 iteration (fewer outer 步骤)|
|Verify a Q-learning / PPO implementation|Compare to DP-optimal V* on a toy 环境|
|Model-based RL (Phase 9 · 10)|Bellman backup on a learned transition 模型|
|Planning in AlphaZero / MuZero|Monte Carlo Tree Search = 异步 Bellman backup|
|Offline RL (CQL, IQL)|Conservative Q-iteration — DP with a penalty on OOD actions|

每个time someone says "the optimal value 函数," they mean "the DP fixed point." When you see `V*` or `Q*` in a paper, picture this 循环.

## 交付成果

Save as `outputs/skill-dp-solver.md`:

```markdown
---
name: dp-solver
description: Solve a small tabular MDP exactly via policy iteration or value iteration. Report convergence behavior.
version: 1.0.0
phase: 9
lesson: 2
tags: [rl, dynamic-programming, bellman]
---

Given an MDP with a known model, output:

1. Choice. Policy iteration vs value iteration. Reason tied to |S|, |A|, γ.
2. Initialization. V_0, starting policy. Convergence sensitivity.
3. Stopping. Sup-norm tolerance ε. Expected number of sweeps.
4. Verification. V*(s_0) computed exactly. Greedy policy extracted.
5. Use. How this baseline will be used to debug/evaluate sampling-based methods.

Refuse to run DP on state spaces > 10⁷. Refuse to claim convergence without a sup-norm check. Flag any γ ≥ 1 on an infinite-horizon task as a guarantee violation.
```

## 练习

1. **Easy.** Run value iteration on the 4×4 GridWorld with `γ ∈ {0.9, 0.99}`. How many sweeps until `max |ΔV| < 1e-6`? Print `V*` as a 4×4 网格.
2. **Medium.** Compare 策略 iteration vs value iteration on the *stochastic* GridWorld (slip 概率 `0.1`). Count: sweeps, wall-clock time, final `V*(0,0)`. Which converges faster in iterations? In wall-clock?
3. **Hard.** Build modified 策略 iteration: in the 评估 步骤, run only `k` sweeps instead of to convergence. Plot `V*(0,0)` 错误 vs `k` for `k ∈ {1, 2, 5, 10, 50}`. What does the 曲线 tell you about the 评估/improvement tradeoff?

## Key Terms

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|策略 iteration|"DP algorithm"|Alternating 评估 (`V^π`) and improvement (greedy `π` w.r.t. `V^π`) until the 策略 stops changing.|
|Value iteration|"Faster DP"|Bellman optimality backup applied in one sweep; converges to `V*` geometrically.|
|Bellman operator|"The recursion"|`(T V)(s) = max_a Σ P (r + γ V(s'))`; a `γ`-contraction in sup-norm.|
|Contraction|"Why DP converges"|Any operator `T` with `\|\|T x - T y\|\|≤ γ \|\|x - y\|\|` has a unique fixed point.|
|GPI|"Everything is DP"|Generalized 策略 Iteration: any method driving `V` and `π` to mutual consistency.|
|Synchronous update|"Jacobi-style"|Use old `V` throughout a sweep; cleanly analyzable but slower.|
|In-place update|"Gauss-Seidel-style"|Use `V` as it's being updated; converges faster in practice.|

## 延伸阅读

- [Sutton & Barto (2018). Ch. 4 — Dynamic Programming](http://incompleteideas.net/book/RLbook2020.pdf) — the canonical presentation of 策略 iteration and value iteration.
- [Bertsekas (2019). Reinforcement Learning and Optimal Control](http://www.athenasc.com/rlbook.html) — rigorous treatment of contraction-mapping arguments.
- [Puterman (2005). Markov Decision Processes](https://onlinelibrary.wiley.com/doi/book/10.1002/9780470316887) — modified 策略 iteration and its convergence analysis.
- [Howard (1960). Dynamic Programming and Markov Processes](https://mitpress.mit.edu/9780262582300/dynamic-programming-and-markov-processes/) — the original 策略 iteration paper.
- [Bertsekas & Tsitsiklis (1996). Neuro-Dynamic Programming](http://www.athenasc.com/ndpbook.html) — the bridge from DP to approximate-DP / deep RL used by every subsequent lesson.
