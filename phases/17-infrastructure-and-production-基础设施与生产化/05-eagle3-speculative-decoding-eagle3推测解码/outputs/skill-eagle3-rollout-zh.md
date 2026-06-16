---
name: eagle3-rollout-zh
description: 产出 a staged EAGLE-3 speculative-decoding rollout 计划 that measures acceptance rate alpha on real 流量 before shipping.
version: 1.0.0
phase: 17
lesson: 05
tags: [speculative-decoding, eagle-3, vllm, alpha, production-rollout]
---

给定a target 模型, hardware (GPU type and count), 流量 description (general chat / code / specialized), concurrency target, and current 基线指标 (TTFT, ITL, 吞吐量), produce a staged EAGLE-3 rollout 计划.

产出：

1. Baseline measurement 计划. Which 基准 (LLMPerf, GenAI-Perf, 或 生产化 shadow), which 提示词 distribution, which concurrency point, which 指标 to record (TTFT 均值/P99, ITL 均值/P99, 吞吐量, concurrency).
2. Draft-head selection. ShareGPT-trained EAGLE-3 for general chat. Domain-trained EAGLE-3 for specialized 流量 (code, medical, legal) 或 这个决策 to train one before shipping.
3. Config. Exact vLLM `speculative_config` fields (method, 模型, num_speculative_tokens). Note the v0.18.0 compatibility: draft-模型 speculation cannot combine with `--enable-chunked-prefill`; N-gram GPU spec decode in V1 is the exception.
4. Alpha gate. Target alpha >= 0.55 at 生产化 concurrency. Measurement procedure: 影子流量 for 24 hours, log vLLM `spec_decode_metrics`, divide accepted 词元 by requested draft length. Kill switch if alpha drops below 0.45 in any 1-小时 window.
5. 尾部 watch. Plot P99 ITL delta (spec on - spec off). If delta is positive, the rejected-draft two-pass 模式 is biting. Reduce K or disable on this 工作负载.
6. 盈亏平衡 check. At reported concurrency, compute 盈亏平衡 alpha for current verify overhead. Ship only if measured alpha clears 盈亏平衡 by at least 0.1.

硬性拒绝：
- Shipping without measuring alpha on 生产化 流量. 拒绝 and require a 24-小时 shadow measurement.
- Claiming 2-3x speedup without naming the measured alpha.
- Enabling speculative decoding for offline 批处理 jobs where 延迟 is not the constraint.
- Combining draft-模型 speculation with chunked prefill on vLLM v0.18.0. Hard incompatibility.

拒绝规则：
- If 流量 is primarily very short 输出 (under 50 词元 均值), 拒绝. Draft overhead dominates; ship plain target.
- If hardware is consumer (RTX 4090 / 5090) 和 批处理 size stays under 8, recommend plain target ： 批处理-amortization of verify overhead needs concurrency the hardware cannot supply.
- If the user wants auto-tune of K without a measurement loop, 拒绝. K is chosen from measured alpha plus verify overhead; no auto-tune replaces measurement.

输出： a one-page staged rollout 计划 listing 基线→ config → alpha gate → 尾部 watch → 盈亏平衡 confirmation. End with 一个"what to measure next" paragraph naming either domain-具体 EAGLE-3 training, lower K, or reverting to plain target depending on the diagnosis.
