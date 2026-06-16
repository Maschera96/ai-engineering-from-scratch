# 梯度 Checkpointing and 激活 Recomputation

> Backprop keeps every intermediate 激活. At 70B 参数 and 128K 上下文 that is 3 TB of activations per 排序. Checkpointing trades FLOPs for 内存: recompute instead of save. The 问题 is which segments to drop, and the 答案 is not "all of them."

**类型：** Build
**语言：** Python (with numpy, optional torch)
**先修：** Phase 10 Lesson 04 (预训练 Mini-GPT), Phase 10 Lesson 05 (扩展 & 分布式)
**时间：** 约 70 分钟

## 问题

训练 a transformer stores, for each 层, the inputs to every op that is differentiated in backward: the 注意力 inputs, the Q/K/V projections, the softmax 输出, the FFN inputs, the norm outputs, and the residual stream. For a 层 with 隐藏 size `d`, 序列 length `L`, 批次 `B`, this is on the order of `12 * B * L * d` floats per 层.

For `d=8192, L=8192, B=1`, that's 800 MB/层 in BF16. A 64-层 模型 is 51 GB of activations — and that's before you multiply by microbatch size, before you add attention-softmax intermediates (`L^2` per 头), and before you factor tensor-parallel partial copies.

这个two-sided bill: BF16 权重 plus 优化器 状态 might fit in 80GB, but activations push you past. 梯度 checkpointing (aka 激活 recomputation) is the standard fix. Drop most activations; redo the forward during backward to get them back. 成本: extra FLOPs. Benefit: 内存 drops by the 比例 of checkpoint segments to total 层.

Done naively, checkpointing 成本 roughly 33% more forward-pass FLOPs per 步骤. Done well — selective checkpointing per the "smart selection" of Korthikanti et al. — you save 5x 内存 for under 5% FLOP overhead. And with FP8 matmuls, FSDP offload, and expert-parallel MoE this really matters: you can't afford either the 内存 or the wasted 计算.

## 概念

### What Backward Actually Needs

`output = layer(input)`. Backward wants `grad_input` and `grad_params`. To 计算 them it needs:

- `input` (to 计算 `grad_params = input.T @ grad_output` for linear 层)
- some 激活 derivative intermediates (the derivative of ReLU/GELU/softmax depends on the 激活 value)

这个forward pass stores these automatically in the autograd 图. Every `tensor.retain_grad()` and every op that needs its 输入 retains a 参考.

### Naive Full Checkpointing

Split the network into `N` segments. During forward, store only the *输入* to each segment. When backward needs intermediates, rerun the segment's forward pass to materialize them, then differentiate.

Example: 32-层 transformer split into 32 segments of 1 层 each.

- 内存: 32 layer-inputs (small) vs 32 * (激活 volume per 层) (huge).
- Extra 计算: 1 extra forward per segment, i.e., ~33% more forward FLOPs total (since backward is 2x forward, full 步骤 becomes 1 + 1 + 2 = 4 units instead of 1 + 2 = 3).

这is the original Chen et al. 2016 recipe: one checkpoint every `sqrt(L)` 层 to balance 内存 and 计算. For L=64, that's 8 checkpoints.

### Selective Checkpointing (Korthikanti 2022)

Not all activations 成本 the same. The 注意力 softmax 输出 is `B*L*L*heads` and grows *quadratically* with 序列 length. The FFN 隐藏 激活 is `B*L*4d` and grows linearly. For long sequences the softmax dominates.

Selective checkpointing keeps the cheap-to-store activations (linear projections, residuals) and recomputes only the expensive ones (注意力). You pay minimal FLOPs to recompute but save the O(L^2) 内存.

Megatron-Core implements this as "selective" 激活 recomputation. Used in most 2024+ frontier 训练 runs.

### Offload

替代方案 to recompute: ship activations to CPU RAM between forward and backward. Requires PCIe bandwidth; beneficial when idle bandwidth exceeds the 成本 of rematerialization. Mixed strategies are common: checkpoint some 层, offload others.

FSDP2 ships offload as a first-class option. Offload shines when GPU is bottlenecked on 内存 but CPU-GPU transfer has headroom.

### Recompute 成本 模型

Per-step FLOPs with naive checkpointing every `k` 层 out of `L`:

```text
flops_fwd_normal = L * f_layer
flops_bwd_normal = 2 * L * f_layer
flops_total_normal = 3 * L * f_layer

flops_fwd_ckpt = L * f_layer
flops_recompute = L * f_layer  # one extra forward per layer in the segment
flops_bwd_ckpt = 2 * L * f_layer
flops_total_ckpt = 4 * L * f_layer
overhead = 4 / 3 - 1 = 0.33 = 33%
```

With selective checkpointing you recompute only the 注意力 kernel, not the whole 层:

```text
flops_recompute_selective = L * f_attention ~= L * f_layer * 0.15
overhead_selective = (3 + 0.15) / 3 - 1 = 0.05 = 5%
```

### 内存 Savings 模型

激活 volume per 层: `A`. For `L` 层, total 激活 内存: `L * A`.

Full checkpoint (segment size 1): store only `L * input_volume` (~`L * 1/10 A` for a standard transformer). Saves ~`9 * L * A * 1/10`.

Checkpoint every `k` 层: store `L/k * A` plus `k-1` 层' worth within the active segment.

At `k = sqrt(L)`, 内存 and recompute 成本 both 规模 with `sqrt(L)` — the optimal tradeoff for uniform-cost 层.

### When Not to Checkpoint

- 这个innermost 层 of a 流水线 stage already in-flight. They have to finish anyway.
- 这个first and last 层 if they dominate the stage's 计算 (rare in transformers).
- 注意力 kernels already using FlashAttention — Flash already recomputes the softmax fast, so additional layer-level checkpointing adds little on top.

### Implementation Patterns

1. **函数 wrapper:** wrap a segment in `torch.utils.checkpoint.checkpoint(fn, input)`. PyTorch stores only `input`, recomputes everything else on backward.

2. **Decorator-based:** 标签 层 as checkpointable; the trainer decides at 配置 time which segments get wrapped.

3. **Manual explicit recompute:** write the backward pass yourself, calling a custom `recompute_forward` that duplicates the forward with the stored 输入.

All three give the same functional result. Wrappers are the standard idiom.

### Interaction with TP / PP / FP8

- **Tensor 并行:** checkpoint inputs must be gathered or rescattered on recompute; handle the communication 成本.
- **流水线 并行:** typical pattern is to checkpoint each pipeline-stage's forward so reverse-order microbatches can reuse 激活 内存.
- **FP8 recompute:** amax histories updated during recompute must match the original forward's, or the FP8 规模 drifts. Most frameworks snapshot the 规模.

## 动手构建

### 步骤 1: A Toy 模型 With Segments

```python
import numpy as np


def linear_forward(x, w, b):
    return x @ w + b


def relu(x):
    return np.maximum(x, 0)


def layer_forward(x, w1, b1, w2, b2):
    h = relu(linear_forward(x, w1, b1))
    return linear_forward(h, w2, b2)


def model_forward(x, params):
    activations = [x]
    h = x
    for w1, b1, w2, b2 in params:
        h = layer_forward(h, w1, b1, w2, b2)
        activations.append(h)
    return h, activations
```

### 步骤 2: Naive Backward Needing All Activations

```python
def model_backward(grad_output, activations, params):
    grads = [None] * len(params)
    g = grad_output
    for i in range(len(params) - 1, -1, -1):
        w1, b1, w2, b2 = params[i]
        x_in = activations[i]
        h_pre = linear_forward(x_in, w1, b1)
        h = relu(h_pre)
        gh = g @ w2.T
        gw2 = h.T @ g
        gb2 = g.sum(axis=0)
        g_pre = gh * (h_pre > 0)
        gx = g_pre @ w1.T
        gw1 = x_in.T @ g_pre
        gb1 = g_pre.sum(axis=0)
        grads[i] = (gw1, gb1, gw2, gb2)
        g = gx
    return g, grads
```

### 步骤 3: Checkpoint-Every-k 内存

```python
def model_forward_checkpointed(x, params, k=4):
    saved_inputs = [x]
    h = x
    for i, (w1, b1, w2, b2) in enumerate(params):
        h = layer_forward(h, w1, b1, w2, b2)
        if (i + 1) % k == 0:
            saved_inputs.append(h)
    return h, saved_inputs


def model_backward_checkpointed(grad_output, saved_inputs, params, k=4):
    grads = [None] * len(params)
    g = grad_output
    segments = [(j * k, min((j + 1) * k, len(params))) for j in range(len(saved_inputs))]
    for seg_idx in range(len(saved_inputs) - 1, -1, -1):
        start, end = segments[seg_idx]
        if start >= end:
            continue
        x_in = saved_inputs[seg_idx]
        _, seg_acts = model_forward(x_in, params[start:end])
        g, seg_grads = model_backward(g, seg_acts, params[start:end])
        for j, gr in enumerate(seg_grads):
            grads[start + j] = gr
    return g, grads
```

### 步骤 4: 成本 模型

```python
def checkpoint_cost(n_layers, segment_size, flops_per_layer=1.0):
    fwd = n_layers * flops_per_layer
    recompute = n_layers * flops_per_layer
    bwd = 2 * n_layers * flops_per_layer
    return {
        "fwd": fwd,
        "recompute": recompute,
        "bwd": bwd,
        "total": fwd + recompute + bwd,
        "overhead_vs_no_ckpt": (fwd + recompute + bwd) / (fwd + bwd) - 1.0,
    }


def selective_checkpoint_cost(n_layers, attention_fraction=0.15,
                              flops_per_layer=1.0):
    fwd = n_layers * flops_per_layer
    recompute = n_layers * attention_fraction * flops_per_layer
    bwd = 2 * n_layers * flops_per_layer
    return {
        "fwd": fwd,
        "recompute": recompute,
        "bwd": bwd,
        "total": fwd + recompute + bwd,
        "overhead_vs_no_ckpt": (fwd + recompute + bwd) / (fwd + bwd) - 1.0,
    }
```

### 步骤 5: 内存 Estimator

```python
def activation_memory_mb(n_layers, hidden=8192, seq=8192,
                        batch=1, bytes_per_value=2):
    per_layer = 12 * batch * seq * hidden * bytes_per_value
    return n_layers * per_layer / 1e6


def memory_after_checkpoint(n_layers, segment_size, hidden=8192,
                           seq=8192, batch=1, bytes_per_value=2):
    n_seg = max(1, n_layers // segment_size)
    saved = (n_seg + segment_size) * 1 * batch * seq * hidden * bytes_per_value
    return saved / 1e6
```

### 步骤 6: Optimal Segment Size

```python
def optimal_segment(n_layers):
    return int(round(np.sqrt(n_layers)))
```

### 步骤 7: Selective Checkpoint Decision

```python
def should_recompute(layer_type, activation_bytes, recompute_flops_ratio):
    if layer_type == "attention" and activation_bytes > 100 * 1e6:
        return True
    if layer_type == "ffn" and activation_bytes > 500 * 1e6:
        return recompute_flops_ratio < 0.1
    return False
```

## 实际使用

- **torch.utils.checkpoint**: `from torch.utils.checkpoint import checkpoint` — the canonical wrapper in PyTorch. Wraps a 函数; stores only inputs, recomputes on backward.
- **Megatron-Core 激活 recomputation**: supports `selective`, `full`, and `block` modes. Standard in 2024+ frontier 训练.
- **FSDP2 offload**: `module.to_empty(device="cpu")` with `offload_policy` in FSDP2 shards activations to CPU instead of recomputing.
- **DeepSpeed ZeRO-Offload**: CPU offload for 优化器 states and activations, complementing checkpointing.

## 交付成果

这lesson produces `outputs/prompt-activation-recompute-policy.md` — a 提示词 that takes your 模型 配置 (层, 隐藏, seq, 批次) and available GPU 内存 and emits a per-layer recompute 策略 (none / selective / full / offload).

## 练习

1. Verify correctness. Run `model_forward` + `model_backward` (full activations) vs `model_forward_checkpointed` + `model_backward_checkpointed` (segments). 参数 gradients must be identical to machine precision.

2. Sweep segment size `k` from 1 to `L`. Plot FLOP overhead and 内存. Find the knee of the 曲线.

3. Implement selective checkpointing: store the attention-module 输入 but not its intermediates. Measure the FLOP overhead vs full-layer checkpointing for a 32-层 模型 at seq=8192.

4. Add offload. Save segment inputs to a simulated "CPU buffer" (a separate list). Measure "PCIe bandwidth" as bytes/time and find the breakeven point between offload and recompute.

5. 基准 a 真实 PyTorch transformer with and without `torch.utils.checkpoint`. Measure 内存 (via `torch.cuda.max_memory_allocated`) and 步骤 time.

## Key Terms

|Term|What people say|What it actually means|
|------|----------------|----------------------|
|梯度 checkpointing|"Save 内存 by redoing forward"|Store segment inputs only; recompute intermediates during backward to get gradient-support tensors|
|激活 recomputation|"Same as checkpointing"|The HPC-flavored name for the same technique|
|Segment size (k)|"How many 层 per checkpoint"|Number of 层 whose intermediates are dropped and rematerialized together|
|Selective checkpointing|"Korthikanti's trick"|Recompute only expensive-to-store activations (注意力 softmax); keep cheap ones|
|Full checkpointing|"The naive version"|Recompute every 层's intermediates in every segment|
|块 checkpointing|"Coarse-grained"|Checkpoint whole transformer 块; largest granularity|
|FLOP overhead|"The 计算 tax"|Extra FLOPs per 步骤 = (recompute FLOPs) / (fwd + bwd FLOPs); 33% naive, 5% selective|
|激活 offload|"Ship to CPU"|Move activations to CPU RAM across forward->backward; 替代方案 to recompute|
|sqrt-L rule|"The classical optimum"|For uniform-cost 层, optimal checkpoint spacing is sqrt(L) 层|
|Attention-softmax volume|"The O(L^2) problem"|L^2 * 头 * 批次 floats; dominates 激活 内存 at long contexts|

## 延伸阅读

- [Chen et al., 2016 -- "Training Deep Nets with Sublinear Memory Cost"](https://arxiv.org/abs/1604.06174) -- the original paper that formalized 梯度 checkpointing
- [Korthikanti et al., 2022 -- "Reducing Activation Recomputation in Large Transformer Models"](https://arxiv.org/abs/2205.05198) -- selective 激活 recomputation and the formal 成本 analysis
- [Pudipeddi et al., 2020 -- "Training Large Neural Networks with Constant Memory using a New Execution Algorithm"](https://arxiv.org/abs/2002.05645) -- 替代方案 constant-memory approach via reverse-mode rematerialization
- [Ren et al., 2021 -- "ZeRO-Offload: Democratizing Billion-Scale Model Training"](https://arxiv.org/abs/2101.06840) -- 激活 offload at 规模
- [PyTorch torch.utils.checkpoint docs](https://pytorch.org/docs/stable/checkpoint.html) -- the standard API
- [Megatron-Core activation recomputation documentation](https://docs.nvidia.com/nemo-framework/user-guide/latest/nemotoolkit/features/memory_optimizations.html) -- selective, full, and 块 modes
