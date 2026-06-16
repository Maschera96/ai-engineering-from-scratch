---
name: parallel-inference-router-zh
description: Route a 推理 workload between voting, tree-of-thought, multi-agent, Hogwild!, and 推测解码 strategies.
version: 1.0.0
phase: 10
lesson: 22
tags: [parallel-inference, hogwild, speculative-decoding, tree-of-thought, multi-agent, reasoning]
---

给定a 推理 workload profile (词元 预算 per 任务, 任务 parallelism characteristics, 模型 family, deployment 目标, 延迟 预算), recommend a parallel-推理 strategy or combination.

Produce:

1. 任务 分类. Long 推理 (5k+ 词元), medium chain-of-thought (1k-5k), short chat (under 1k), or 分类. Drives the first-pass decision.
2. Parallelism axis. Within-sequence (推测解码) vs across-sequence (voting, Hogwild!, multi-agent). Most workloads benefit from the within-sequence axis first.
3. Strategy recommendation. Pick from: 推测解码 only (safe default for any workload above 100 词元), speculative + Hogwild! (long 推理 with parallelizable structure), tree-of-thought (explicit branch-and-prune problems), multi-agent (role-specialization problems), voting ensemble (high-stakes 分类).
4. 参数 settings. For 推测解码: draft family (EAGLE-3 default) and `N` (Phase 10 · 15 skill). For Hogwild!: worker count N (2 to 4, rarely more), coordination 提示词 template, single-node deployment confirmation.
5. Combined speedup estimate. If combining 推测解码 with Hogwild!, report the multiplicative speedup (typical range: 3x spec * 1.5-2x Hogwild! = 4.5-6x).

Hard rejects:
- Hogwild! for any workload under 2000 词元. Coordination overhead dominates.
- Hogwild! on non-reasoning 模型 (no emergent coordination).
- Multi-agent framework for problems that do not have a natural role decomposition.
- Tree-of-thought without explicit branch-and-prune logic (the strategy reduces to linear CoT otherwise).
- Running Hogwild! across nodes (cross-node 缓存 synchronization is too slow).

Refusal rules:
- 如果the workload is experimental research, recommend Hogwild! as an experiment rather than a 生产 bet. The speedups are task-dependent and real-world deployment is rare as of April 2026.
- 如果the 用户 asks for guaranteed speedup, refuse and explain that only 推测解码 has the strong-guarantee property (输出 分布 preserved). Hogwild! is empirical.
- 如果the 用户 has limited VRAM, refuse Hogwild! N>2 — each worker needs its own 激活 内存 even though the 缓存 is shared.

输出： a one-page recommendation listing 任务 分类, parallelism axis, strategy, 参数, and combined speedup estimate. End with a "rollback trigger" paragraph naming the specific 延迟 or accuracy 指标 that would justify reverting to 推测解码 alone if Hogwild! does not pay off in the first 100 生产 requests.
