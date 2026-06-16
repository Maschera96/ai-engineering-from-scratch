---
name: llm-pipeline-reviewer-zh
description: Review an end-to-end LLM 训练 流水线 manifest before a multi-million-dollar run.
version: 1.0.0
phase: 10
lesson: 13
tags: [pipeline, training, manifest, eval-gate, cost, rollback]
---

给定a proposed 训练 流水线 manifest (YAML or JSON describing 分词器, 数据, 预训练, SFT, 对齐, 评估, 量化, and serving stages), produce a review covering:

1. Stage 图. Confirm each stage has typed inputs and outputs. Call out missing dependencies, implicit 状态, or any stage that consumes a bare directory instead of a named 工件 hash.
2. Hash 链. Verify output_hash of stage N equals one of the input_hashes of every downstream stage. Any mismatch means the manifest is incoherent and the 流水线 must not start.
3. 评估 gate. Every 指标 in the gate list must be numeric, have an operator, a 阈值, and a measurement 来源. Reject any gate that is subjective ("looks good"), unbounded (no 阈值), or measured on the 训练 数据.
4. 回归 guard. The new 模型's core benchmarks (MMLU, MATH, HumanEval+, GPQA, or a domain-specific equivalent) must have 基线 numbers attached. A run with no baselines is a run with no 回归 detection.
5. KL 预算. 对齐 stages (RLHF, DPO, CAI, GRPO) must declare a cumulative KL cap against the 参考. Unbounded KL is an unbounded drift.
6. Contamination check. 训练 数据 shards and 评估 sets must have a documented overlap check (exact match or 13-gram). Required pass 阈值: <0.1%.
7. 成本 estimate. Pre-run estimate for each stage plus a total, compared against the 预算 gate. If estimate > 预算, 流水线 refuses to start.
8. Rollback plan. For each stage, named actions on failure: re-run, fall back to previous 工件, revise inputs and re-run downstream. Expensive stages (预训练) must have a warm checkpoint strategy.
9. 工件 store. Checkpoints, datasets, 分词器s, 评估 reports must be content-addressed (SHA-256). Filename-addressed 工件 ("latest.pt") are a hard reject.
10. Observability. Every stage must emit 结构化 logs with a trace ID, stage name, 输入 hashes, 输出 hash, wall clock, and 成本. Missing trace IDs mean the run cannot be debugged after the fact.

Red flags that halt the review:
- a gate missing a measurement 来源 (gate on a 指标 no stage computes)
- a stage that shares a checkpoint with a downstream stage (no separation of concerns)
- an 对齐 stage with no 参考 模型 (no anchor for KL)
- an LLM-as-judge 评估 where the judge is the same 模型 family as the 策略 (contamination)
- a 成本 estimate that exceeds the 预算 by more than 20%
- a rollback plan consisting solely of "re-run from scratch"

输出： a two-page review with PASS/HOLD per gate, the exact manifest field or missing field that produced each verdict, and the minimum change required to flip a HOLD into a PASS.
