---
name: prompt-gpt-architecture-analyzer-zh
description: Analyze 架构 choices in any GPT-style transformer 模型
version: 1.0.0
phase: 10
lesson: 4
tags: [gpt, transformer, architecture, attention, kv-cache, scaling, pre-training]
---

# GPT 架构 Analyzer

当evaluating a GPT-style 模型 from a technical report, 模型 card, or 训练 log, use this framework to break down the 架构 and identify design 取舍.

## Analysis 协议

### 1. 参数 Allocation Breakdown

计算 the exact 参数 count for each component:

- **词元 嵌入s**: vocab_size x embed_dim
- **Position 嵌入s**: max_seq_len x embed_dim
- **Per-block 注意力**: 4 x embed_dim x embed_dim (Q, K, V, 输出 projections)
- **Per-block FFN**: 2 x embed_dim x ff_dim + embed_dim + ff_dim (two linear 层 + biases)
- **Per-block LayerNorm**: 4 x embed_dim (two norms, each with 规模 + 偏差)
- **Final LayerNorm**: 2 x embed_dim
- **输出 头**: vocab_size x embed_dim (or 0 if weight-tied with 词元 嵌入s)

标记if any single component exceeds 40% of total 参数. The 嵌入 matrix dominates in small 模型. 注意力 and FFN dominate in large 模型.

### 2. 注意力 Design Analysis

Evaluate the 注意力 configuration:

- **头 维度**: embed_dim / num_heads. Standard is 64 (GPT-2) or 128 (Llama 3). Below 32 limits per-head expressiveness. Above 128 wastes 计算 with little benefit.
- **头 per 层**: More 头 = more diverse 注意力 patterns but more 内存 for KV 缓存.
- **Grouped 查询 注意力 (GQA)**: Does the 模型 share K/V 头 across multiple Q 头? Llama 3 uses GQA with 8 KV 头 for 32 Q 头. This reduces KV 缓存 by 4x.
- **上下文 length**: Max position 嵌入s. RoPE allows extrapolation beyond 训练 length. Absolute position 嵌入s do not.

### 3. 内存 预算

For 推理 at the 模型's maximum 上下文 length:

- **权重 (FP16)**: total_params x 2 bytes
- **KV 缓存 (FP16)**: 2 x num_layers x num_kv_heads x head_dim x max_seq_len x 2 bytes
- **Activations**: batch_size x seq_len x embed_dim x 2 bytes x num_layers (approximate)

标记if KV 缓存 exceeds 权重 内存. This happens for long-context 模型 (128K+) and indicates the 模型 is memory-bound during decode.

### 4. 计算 Profile

- **Prefill FLOPS per 词元**: approximately 2 x total_params (one matmul per 参数, forward pass)
- **Decode FLOPS per 词元**: same as prefill but on a single 词元
- **Prefill bottleneck**: compute-bound (GPU TFLOPS)
- **Decode bottleneck**: memory-bound (GPU 内存 bandwidth)
- **Arithmetic intensity**: FLOPS per byte of 内存 accessed. Below 100 = memory-bound.

### 5. 扩展 Decisions

Evaluate against known 扩展 laws:

- **Chinchilla optimal**: For a given 计算 预算 C, optimal 模型 size N and 词元 count D satisfy N ~ D (roughly equal 扩展). A 7B 模型 needs ~140B 词元.
- **Llama 3 overtrained**: Meta 训练后的 Llama 3 8B on 15T 词元 (100x Chinchilla optimal). Overtraining small 模型 on more 数据 produces better per-token 推理 成本.
- **宽度 vs 深度**: Deeper 模型 (more 层) are generally more sample-efficient than wider 模型 (larger embed_dim) for the same 参数 count.

## Red Flags

- **FFN 比例 not 4x**: Standard is ff_dim = 4 x embed_dim. Llama uses 8/3 x embed_dim with SwiGLU. Deviations should be justified.
- **No 权重 tying**: The 输出 头 should share 权重 with 词元 嵌入s unless vocab_size is very large relative to embed_dim.
- **No GQA above 13B**: 模型 above 13B without grouped-query 注意力 will have excessively large KV caches.
- **No RoPE for long 上下文**: Absolute position 嵌入s do not extrapolate beyond 训练 length. 模型 targeting 32K+ 上下文 should use rotary 嵌入s.
- **学习 速率 too high for 模型 size**: Larger 模型 need lower peak 学习 rates. GPT-2 Small uses 6e-4. Llama 3 405B uses 8e-5.

## 输出格式

1. **参数 Table**: component-by-component 参数 counts with percentages
2. **内存 预算**: 权重, KV 缓存, and 激活 内存 at max 上下文 length
3. **计算 Profile**: prefill and decode throughput estimates for A100/H100
4. **Design Assessment**: what the 模型 gets right and what is non-standard
5. **扩展 Verdict**: whether the 模型 is appropriately sized for its 训练 数据
