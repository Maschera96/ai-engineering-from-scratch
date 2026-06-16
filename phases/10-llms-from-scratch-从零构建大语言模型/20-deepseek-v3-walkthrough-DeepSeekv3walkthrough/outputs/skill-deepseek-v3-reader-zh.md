---
name: deepseek-v3-reader-zh
description: Read a DeepSeek-family 配置 and produce a component-by-component 架构 analysis.
version: 1.0.0
phase: 10
lesson: 20
tags: [deepseek-v3, deepseek-r1, mla, moe, mtp, dualpipe, architecture]
---

给定a DeepSeek-family 模型 (V3, R1, or any derivative) and its 配置 (hidden_size, 层, num_experts, kv_lora_rank, etc.), produce an 架构 analysis that breaks the 模型 down by component and identifies which DeepSeek-specific innovations it uses.

Produce:

1. Field-by-field 配置 read. For each field, name the component it maps to and the 参数 count it contributes. Format: `field_name: value → interpretation → parameter contribution`.
2. 参数 breakdown. Total 参数, active 参数, active 比例. Split by 嵌入, per-layer 注意力, per-layer MLP (稠密 vs expert), 路由器, MTP module, LM 头, RMSNorm total.
3. KV 缓存 at 目标 上下文. Report BF16 and FP8 values. Include a comparison to a Llama-3-style GQA(8/128) 基线 at the same 上下文 and 隐藏 size.
4. Innovation checklist. For each of MLA, MTP, aux-loss-free 路由, DualPipe, identify whether the 模型 uses it and where in the 配置/paper this is visible.
5. Sanity check. 计算 the 模型's 推理 内存 预算 (权重 + KV 缓存 + activations) on a specific deployment 目标 (H100 80GB, H200 141GB, MI300X 192GB, single 节点 vs multi-node). Report whether it fits and what 量化 would be needed.

Hard rejects:
- Any analysis that conflates DeepSeek-V3 with GPT-class 稠密 模型. The 架构 is materially different.
- Claiming MLA is faster than GQA without specifying 上下文 length. At short 上下文 (under 4k) they are comparable; MLA wins at long 上下文.
- Interpreting MTP as a replacement for 推测解码. It is a 预训练 目标 that also doubles as a draft.

Refusal rules:
- 如果the provided 配置 is missing `kv_lora_rank`, `num_experts`, or `first_k_dense_layers`, refuse — this is not a DeepSeek-family 模型.
- 如果the 用户 asks for the exact published 参数 count match (to the nearest 100M), refuse and explain that the published number includes implementation-specific structural 参数 a simplified calculator does not exactly reproduce. Direct them to the paper's Section 2 appendix.
- 如果the 目标 deployment 目标 is a consumer GPU (24GB or less), refuse and recommend a 量化的 distilled DeepSeek-family derivative instead.

输出： a one-page 架构 analysis listing fields, 参数 breakdown, KV 缓存, innovation checklist, and deployment fit. End with a "what to read next" paragraph naming one of NSA (Phase 10 · 17), MLA ablations from the V2 paper, or the V3 technical report's Section 2 appendix, depending on what 问题 the analysis surfaced.
