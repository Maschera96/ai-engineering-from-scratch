---
name: hybrid-picker-zh
description: 选择between pure Transformer, Jamba-style hybrid, and pure SSM for a given workload.
version: 1.0.0
phase: 10
lesson: 21
tags: [jamba, mamba, ssm, hybrid, long-context, memory-budget, architecture]
---

给定a workload specification (上下文 length profile p50/p99, 任务 mix, 内存 预算 per GPU, 目标 throughput, quality-vs-speed priority), recommend between a pure Transformer (+MoE +MLA), a Jamba-style hybrid, and a pure Mamba 模型.

Produce:

1. Context-length bucket. Short (under 16k), medium (16k-64k), long (64k-256k), or ultra-long (256k-plus). Drives the first-pass decision.
2. 架构 recommendation. Pick one of pure Transformer, 1:7 hybrid, 1:3 hybrid, 1:15 hybrid, or pure Mamba. Justify using the 上下文 bucket plus the 任务's in-context-recall demands.
3. 内存 预算 check. 计算 KV 缓存 + SSM 状态 at 目标 上下文. Confirm it fits on the 目标 accelerator after accounting for 权重 and 激活 内存 (typically 10-20 GB on top of 权重 and KV 缓存).
4. 质量 tradeoff disclosure. 文档 the 质量 成本 of the chosen sparsity level. Hybrids below 1:7 比例 degrade on in-context 检索 by measurable amounts; pure Mamba fails on some state-tracking tasks.
5. 推理 stack compatibility. Confirm the chosen 架构 is supported by the 目标 stack (vLLM, TensorRT-LLM, SGLang, llama.cpp). Hybrids have thinner tooling coverage than pure Transformers.

Hard rejects:
- Jamba-style hybrid for 上下文 under 16k. The architectural overhead is not justified.
- Pure Mamba for reasoning-heavy or multi-document cross-reference tasks. State-tracking limits bite.
- Sub-1:15 hybrid ratios. Below this, in-context recall is unreliable.
- Any recommendation that does not fit the computed 内存 预算 on the specified accelerator.

Refusal rules:
- 如果the workload is genuinely mixed short and long 上下文, refuse the hybrid recommendation and recommend the pure Transformer (with MLA if possible) — hybrids shine on long-context workloads specifically.
- 如果the accelerator is consumer-grade (24GB or less), refuse hybrid-size 模型 and recommend a distilled small hybrid or a 量化的 pure Transformer.
- 如果the workload is latency-sensitive batch-1 生成 and the 模型 is new (no existing deployment path), refuse and recommend a well-supported pure Transformer with 推测解码 (Phase 10 · 15) as the simpler path.

输出： a one-page recommendation listing 上下文 bucket, 架构 choice, KV 缓存 at 目标 上下文, 质量 tradeoff disclosure, and 推理 stack compatibility. End with a "what to monitor" paragraph naming the specific long-context 评估 (RULER, LongBench, needle-in-haystack) that would confirm the recommendation in the first 10k 生产 requests.
