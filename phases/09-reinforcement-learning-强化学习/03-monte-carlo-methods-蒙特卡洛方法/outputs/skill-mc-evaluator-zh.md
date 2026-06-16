---
name: mc-evaluator-zh
description: Evaluate a 策略 via Monte Carlo rollouts and produce a convergence report with DP-comparison if available.
version: 1.0.0
phase: 9
lesson: 3
tags: [rl, monte-carlo, evaluation]
---

给定an 环境 (episodic, with reset+步骤 API) and a 策略, 输出：

1. Method. First-visit vs every-visit MC. 原因.
2. Episode 预算. 目标 number, 方差 diagnostic, expected standard 错误.
3. Exploration plan. ε 调度 (if needed) or exploring starts.
4. Gold-standard comparison. DP-optimal V* if tabular; otherwise a bound from a Q-learning / PPO 基线.
5. Termination check. Max-step cap, timeouts, handling of non-terminating trajectories.

拒绝 run MC on non-episodic tasks without a finite horizon cap. 拒绝 report V^π estimates from fewer than 100 episodes per 状态 for tabular tasks. 标记任何 策略 with zero-variance actions as an exploration 风险.
