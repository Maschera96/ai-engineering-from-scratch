---
name: skill-inference-optimization-zh
description: 诊断 and 优化 LLM 推理 serving throughput, 延迟, and 成本
version: 1.0.0
phase: 10
lesson: 12
tags: [inference, kv-cache, batching, speculative-decoding, vllm, optimization]
---

# LLM 推理 优化 Pattern

Two phases: prefill (compute-bound, 并行) and decode (memory-bound, sequential).
每个优化 目标 one or both.

```text
Request -> Prefill (process prompt) -> Decode (generate tokens) -> Response
              |                            |
         Compute-bound               Memory-bound
         Optimize: fusion,           Optimize: batching,
         prefix caching              quantization, speculation
```

## Decision framework

### 步骤 1: Identify your bottleneck

Measure ops:byte 比例 for your workload:

|ops:byte|Bound|What to 优化|
|----------|-------|-----------------|
|< 50|内存|Quantize KV 缓存, increase 批次 size|
|50-200|Transitional|Both matter, start with batching|
|> 200|计算|Kernel fusion, tensor parallelism, FP8|

### 步骤 2: Pick your engine

- **Default**: vLLM (widest 模型 support, PagedAttention, OpenAI-compatible API)
- **Multi-turn / 结构化 输出**: SGLang (RadixAttention prefix 缓存, constrained decoding)
- **Max NVIDIA throughput**: TensorRT-LLM (kernel fusion, FP8 on H100)

### 步骤 3: Apply optimizations in order

1. **KV 缓存** -- always on, no downside
2. **Continuous batching** -- always on, no downside (vLLM/SGLang do this by default)
3. **Prefix 缓存** -- enable if you have shared 系统 prompts (most chatbots do)
4. **量化** -- KV 缓存 INT8/FP8 reduces 内存 2-4x with minimal 质量 损失
5. **Speculative decoding** -- add when 延迟 matters more than throughput
6. **Tensor parallelism** -- split across GPUs when 模型 does not fit on one

## KV 缓存 内存 formula

```text
per_token = 2 * num_layers * num_kv_heads * head_dim * bytes_per_param
total = per_token * sequence_length * num_concurrent_users
```

Quick 参考 for common 模型 (BF16):

|模型|Per 词元|100 users @ 4K|
|-------|-----------|----------------|
|Llama 3 8B|32 KB|12.5 GB|
|Llama 3 70B|320 KB|125 GB|
|Llama 3 405B|504 KB|197 GB|

## Speculative decoding checklist

- Draft 模型 should be 5-10x smaller than 目标 (e.g., 8B drafts for 70B)
- Acceptance 速率 > 70% for meaningful speedup
- Best on predictable 文本 (code, 结构化 输出, natural language)
- Worst on creative/sampling-heavy tasks (low temperature helps)
- EAGLE > draft-target > n-gram for most workloads

## Common mistakes

- Running decode at 批次=1 (memory-bound, GPU 95% idle on 计算)
- Allocating contiguous KV 缓存 块 (use PagedAttention, get near-zero waste)
- Ignoring prefix 缓存 when 80% of requests share the same 系统 提示词
- Over-provisioning GPU 内存 for 模型 权重, leaving nothing for KV 缓存
- Measuring throughput without measuring 延迟 (high throughput at 10s TTFT is useless)
- Using 推测解码 with high temperature (acceptance 速率 drops below 50%)

## Monitoring checklist

- 时间 to first 词元 (TTFT): prefill 延迟, 目标 < 500ms for interactive use
- Inter-token 延迟 (ITL): decode speed, 目标 < 50ms for streaming
- Throughput (词元/second): total across all concurrent users
- KV 缓存 utilization: percentage of allocated 缓存 in use
- 批次 utilization: percentage of 批次 slots filled per iteration
- Queue 深度: requests waiting for a 批次 slot
