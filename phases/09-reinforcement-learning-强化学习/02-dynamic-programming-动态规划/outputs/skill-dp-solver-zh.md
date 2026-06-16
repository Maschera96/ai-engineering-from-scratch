---
name: dp-solver-zh
description: Solve a small tabular MDP exactly via 策略 iteration or value iteration. Report convergence behavior.
version: 1.0.0
phase: 9
lesson: 2
tags: [rl, dynamic-programming, bellman]
---

给定an MDP with a known 模型, 输出：

1. Choice. 策略 iteration vs value iteration. 原因 tied to |S|, |A|, γ.
2. Initialization. V_0, starting 策略. Convergence sensitivity.
3. Stopping. Sup-norm tolerance ε. Expected number of sweeps.
4. Verification. V*(s_0) computed exactly. Greedy 策略 extracted.
5. Use. How this 基线 will be used to 调试/evaluate sampling-based methods.

拒绝 run DP on 状态 spaces > 10⁷. 拒绝 claim convergence without a sup-norm check. 标记任何 γ ≥ 1 on an infinite-horizon 任务 as a guarantee violation.
