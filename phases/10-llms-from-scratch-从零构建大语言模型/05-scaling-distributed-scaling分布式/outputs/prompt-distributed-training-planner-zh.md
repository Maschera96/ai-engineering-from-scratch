---
name: prompt-distributed-training-planner-zh
description: Plan a 分布式 训练 run given 模型 size and available hardware
version: 1.0.0
phase: 10
lesson: 5
tags: [distributed-training, fsdp, deepspeed, tensor-parallelism, pipeline-parallelism, scaling]
---

# 分布式 训练 Planner

当planning a 分布式 训练 run for a large 语言模型, use this framework to determine the parallelism strategy, 内存 预算, communication overhead, and expected throughput.

## 输入 Requirements

Provide:
- **模型 size** (参数 in billions)
- **目标 训练 词元** (in trillions)
- **Available GPUs** (type: A100/H100/H200, count, interconnect: NVLink/InfiniBand)
- **GPU 内存** (80GB for A100/H100, 141GB for H200)
- **Nodes** (GPUs per 节点, number of nodes)
- **预算 constraints** (max 成本 in dollars, max wall-clock time)

## 步骤 1: 内存 预算

Calculate per-GPU 内存 for each component:

|Component|Formula|FP16|FP32|
|-----------|---------|------|------|
|权重|params x bytes_per_param|params x 2|params x 4|
|Adam 优化器 (m + v)|params x 4 x 2|8 bytes/param always|8 bytes/param|
|Gradients|params x bytes_per_param|params x 2|params x 4|
|Activations (estimate)|seq_len x 批次 x 隐藏 x 层 x 2|varies|varies|

如果total exceeds GPU 内存, sharding is required. Try in order:
1. ZeRO-1 (shard 优化器 only) -- cheapest communication
2. ZeRO-2 (+ gradients) -- moderate communication
3. FSDP/ZeRO-3 (+ 权重) -- highest communication but maximum 内存 savings
4. Add 激活 checkpointing if activations still too large
5. Add tensor parallelism if a single 层 does not fit on one GPU

## 步骤 2: Parallelism Strategy

### Decision Tree

1. **Does one 层 fit on one GPU?**
   - No: You need tensor parallelism. Set TP = 2, 4, or 8 (within a 节点).
   - Yes: Skip tensor parallelism.

2. **Does the full 模型 (with sharding) fit on GPUs within one 节点?**
   - No: You need 流水线 parallelism. Set PP = number of nodes / groups.
   - Yes: Skip 流水线 parallelism.

3. **How many remaining GPUs for 数据 parallelism?**
   - DP = total_gpus / (TP x PP)

4. **What sharding level within the 数据 并行 group?**
   - Start with FSDP (ZeRO-3). Reduce to ZeRO-2 or ZeRO-1 if communication is bottleneck.

### Typical Configurations

|模型 Size|Total GPUs|TP|PP|DP|Sharding|
|-----------|-----------|----|----|-----|----------|
|7B|8|1|1|8|FSDP|
|13B|16|2|1|8|FSDP|
|70B|64|8|1|8|FSDP|
|70B|128|8|2|8|FSDP|
|405B|16,384|8|16|128|FSDP|

## 步骤 3: Communication Analysis

Estimate communication volume per 训练 步骤:

- **数据 并行 (all-reduce)**: 2 x gradient_size x (N-1)/N per 步骤
- **FSDP (all-gather + reduce-scatter)**: ~3 x weight_size x (N-1)/N per 步骤 (higher than DP)
- **Tensor 并行 (all-reduce per 层)**: 2 x activation_size x num_layers per 步骤 (needs NVLink)
- **流水线 并行 (point-to-point)**: activation_size per stage boundary (minimal)

如果communication time exceeds 20% of 计算 time, the strategy is communication-bound. Solutions:
- 梯度 accumulation (reduce all-reduce frequency)
- Overlap communication with computation (FSDP does this by default)
- Increase micro-batch size (better compute-to-communication 比例)
- Switch to a less communication-heavy sharding stage

## 步骤 4: Throughput and 成本 Estimate

**FLOPS per 训练 步骤:**
- Forward: ~2 x params x tokens_per_batch
- Backward: ~4 x params x tokens_per_batch (2x forward)
- Total: ~6 x params x tokens_per_batch

**训练 time:**
- total_flops = 6 x params x total_tokens
- time_seconds = total_flops / (num_gpus x gpu_tflops x 1e12 x utilization)
- Typical utilization: 35-45% (accounting for communication, 流水线 bubbles, 内存 overhead)

**成本:**
- total_gpu_hours = num_gpus x time_seconds / 3600
- 成本 = total_gpu_hours x cost_per_gpu_hour

## 步骤 5: 验证 Checklist

Before launching:

1. Per-GPU 内存 fits within hardware 限制 (with 10% headroom)
2. Effective 批次 size matches 目标 (per_gpu_batch x DP x gradient_accumulation_steps)
3. Communication-to-compute 比例 is below 20%
4. 流水线 bubble fraction is below 15% (enough micro-batches)
5. 学习 速率 is scaled for the effective 批次 size
6. Checkpointing frequency accounts for failure 概率 (save every 1-2 小时 for large runs)
7. 梯度 clipping is set (typically 1.0 for large 模型)
8. 预热 步骤 are proportional to total 步骤 (typically 0.1-1% of total)

## Red Flags

- **TP > 8**: Tensor parallelism across nodes (over InfiniBand) is almost always slower than 流水线 parallelism
- **流水线 stages > 32**: Bubble overhead becomes significant even with many micro-batches
- **Effective 批次 size > 10M 词元**: Diminishing returns; may harm convergence
- **Utilization below 30%**: Communication-bound -- re-evaluate parallelism strategy
- **No 激活 checkpointing above 13B**: You will run out of 内存 during the backward pass
- **No 梯度 accumulation with small per-GPU 批次**: 梯度 噪声 increases; accumulate to effective 批次 of 256+ 样本
