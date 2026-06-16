---
name: long-context-eval-zh
description: Design a 长-context 评估 battery 面向 a given 模型 与 use case.
version: 1.0.0
phase: 5
lesson: 28
tags: [nlp, long-context, evaluation]
---

给定一个 target 模型, target context length, 与 use case, 输出:

1. 说明：Tests. NIAH depth × length grid; RULER multi-hop; custom 领域 任务.
2. Sampling. Depths 0, 0.25, 0.5, 0.75, 1.0 at each length.
3. 说明：Metrics. Retrieval pass rate; reasoning pass rate; time-to-first-词元; 成本-per-查询.
4. 说明：Cutoff. Effective 检索 length (90% pass) 与 effective reasoning length (70% pass). Report both.
5. 说明：Regression. 固定 harness, rerun on every 模型 upgrade, surface deltas.

拒绝 to trust a context window 从 the 模型 card alone. 拒绝 NIAH-only 评估 面向 any multi-hop workload. 拒绝 vendor self-reported 长-context scores as independent evidence.
