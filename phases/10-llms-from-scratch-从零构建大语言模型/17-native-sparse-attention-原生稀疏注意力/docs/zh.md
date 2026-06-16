# Native 稀疏 注意力 (DeepSeek NSA)

> At 64k 词元, 注意力 eats 70-80% of decode 延迟. Every open-model lab has a plan to fix it. DeepSeek's NSA (ACL 2025 best paper) is the one that stuck: three 并行 注意力 branches — compressed coarse-grained 词元, selectively retained fine-grained 词元, and sliding windows for local 上下文 — combined through a learned gate. It is hardware-aligned (kernel-friendly), natively trainable (works in 预训练, not bolted on at 推理), and on 64k decodes it runs faster than FlashAttention while 匹配 or beating full 注意力 质量. This lesson builds the three branches end-to-end and shows why the sparsity is end-to-end differentiable.

**类型：** Build
**语言：** Python (stdlib)
**先修：** Phase 7 · 12 (KV 缓存, flash-attention), Phase 7 · 15 (注意力 variants), Phase 10 · 16 (differential 注意力)
**时间：** 约 60 分钟

## 学习目标

- 状态 the three NSA 注意力 branches and what each one captures.
- 解释why NSA is "natively trainable" where 先验 sparse-attention methods were 推理-only.
- 计算 the 注意力 计算 savings of NSA versus full 注意力 at 64k 上下文 as a 函数 of 压缩 块 size and selection top-k.
- Implement the three-branch combination in stdlib Python on a short synthetic 序列 and verify the gating 权重 behave.

## 问题

Full 注意力 at 序列 length N 成本 `O(N^2)` time and `O(N)` KV 缓存 per 层. At 64k 词元, the 计算 and 内存 bandwidth numbers are catastrophic. Measured theoretical estimate from the NSA paper: 注意力 accounts for 70-80% of total decode 延迟 at 64k. Everything downstream — TTFT, 词元/sec, 成本 per million 词元 — is dominated by 注意力 成本.

稀疏 注意力 is the obvious 答案. 先验 attempts fall into two buckets. 修复ed-pattern sparsity (sliding-window, strided, block-local) throws information away and fails on long-range recall tasks. 推理-time sparsity (KV 缓存 pruning, H2O, StreamingLLM) is applied to a 模型 pre-trained on 稠密 注意力 and recovers only a fraction of the potential speedup because the 模型 was never asked to route information through the 稀疏 pattern.

Native 稀疏 注意力 (Yuan et al., DeepSeek + PKU + UW, ACL 2025 best paper, arXiv:2502.11089) does both: a sparsity pattern the 模型 learns during 预训练, implemented as a kernel-aligned algorithm that actually delivers the 计算 savings at 推理. Two years from now, NSA or a direct descendant is the default 注意力 on every frontier long-context 模型.

## 概念

### Three 并行 branches

For each 查询, NSA runs 注意力 three times, against three different views of the KV 缓存:

1. **Compressed branch.** 词元 are grouped into 块 of size `l` (typically 32 or 64). Each 块 is compressed into a single summary 词元 via a small learned MLP. The 查询 attends over these compressed 词元, getting a coarse-grained view of the whole 序列.

2. **Selected branch.** Using 注意力 scores from the compressed branch, the top-k 块 most relevant to the current 查询 are identified. Fine-grained (uncompressed) 词元 from those 块 are read and the 查询 attends over all of them. Think of compressed-branch 注意力 as the 路由 信号 for the selection.

3. **Sliding-window branch.** The 查询 attends to the most recent `W` 词元 (typically 512) for local 上下文. This branch captures the structure-heavy short-range patterns (syntax, local coreference) that the other two might miss.

这个three branch outputs are combined via a learned per-position gate:

```text
out = g_cmp * out_cmp + g_sel * out_sel + g_win * out_win
```

`g_cmp, g_sel, g_win` are gate 权重 from a small MLP on the 查询. They do not have to sum to 1 — they can 权重 branches independently.

### Why this is "natively trainable"

这个selection 步骤 (top-k 块) is discrete. Discrete operations break 梯度 流. 先验 sparse-attention work either skipped backprop through selection (limiting 训练) or used continuous relaxations that did not give 真实 sparsity at 推理.

NSA sidesteps this: the compressed-branch 注意力 IS a differentiable coarse-grained 注意力 on the whole 序列. The top-k operation just reuses the top 注意力 scores from the compressed branch to pick which fine-grained 块 to load. Gradients 流 through the compressed-branch scores (which influence both the compressed 输出 AND the selection logic), and the selected 块' contribution to the final 输出 is also differentiable. The non-differentiable `top_k` operation is a no-op on the forward computational 图 — it only controls which 块 get loaded from 内存.

这is why NSA can be used in 预训练 end to end. The 模型 learns to route information through the three branches jointly, producing a 稀疏 pattern that at 推理 actually delivers the promised speedup.

### Hardware-aligned kernel

NSA's kernel is designed for modern GPU 内存 hierarchies. The kernel loads 查询 by GQA groups (outer 循环), fetches the corresponding 稀疏 KV 块 per group (inner 循环), and runs 注意力 on SRAM. Because each 查询 group sees the same selected 块 (selection is per-query-group, not per-query-head), the KV loads are amortized across the group. Arithmetic intensity stays high.

这个paper reports Triton kernels running 9x faster than FlashAttention on 64k decodes, with the speedup 比例 growing with 序列 length. Forward and backward kernels are both provided.

### The 计算 预算

Let `N` be 序列 length, `l` the 压缩 块 size, `k` the top-k selection count, `w` the sliding window, `b` the selected 块 size (typically equals `l`).

- Compressed branch: `O(N/l)` keys per 查询, so `O(N * N / l)` total.
- Selected branch: `O(k * b)` keys per 查询, so `O(N * k * b)`.
- Sliding branch: `O(w)` keys per 查询, so `O(N * w)`.

Total: `O(N * (N/l + k*b + w))`.

With `N = 64k, l = 64, k = 16, b = 64, w = 512`: per-query 成本 is `1000 + 1024 + 512 = 2536 keys`. Full 注意力 is `64000 keys`. 25x 计算 reduction.

With `N = 128k, l = 64, k = 16, b = 64, w = 512`: per-query 成本 is `2000 + 1024 + 512 = 3536 keys`. Full 注意力 is `128000 keys`. 36x reduction. The benefit grows with 序列 length, which is the whole point.

### How does it compare

|Method|Differentiable|真实 推理 speedup|Long-range recall|
|--------|---------------|----------------------|-------------------|
|Sliding window only|yes|yes|fails|
|Strided / block-sparse|yes|yes|partial|
|KV pruning (H2O, StreamingLLM)|N/A (推理-time)|yes|partial|
|MoBA (Moonshot)|partial|yes|good|
|NSA|yes (natively)|yes (9x at 64k)|matches full 注意力|

MoBA (Moonshot, arXiv:2502.13189) was concurrently published and takes a similar three-is-better-than-one approach, applying the MoE principle to 注意力 块. NSA and MoBA are the two architectures to know for 2026 long-context 预训练.

```figure
sliding-window-attention
```

## 动手构建

`code/main.py` implements the three branches on a short synthetic 序列 and shows:

- 这个压缩 MLP (a simple mean-pool 基线 is used for pedagogical clarity; the 真实 NSA uses a learned MLP).
- 这个top-k 块 selection driven by compressed-branch scores.
- 这个sliding-window 注意力 on the last `w` 词元.
- 这个gated combination.
- 一个compute-count printout comparing to full 注意力.

### 步骤 1: compress 词元 into 块

```python
def compress(K, l):
    n = len(K)
    n_blocks = (n + l - 1) // l
    out = []
    for b in range(n_blocks):
        start, end = b * l, min((b + 1) * l, n)
        block = K[start:end]
        summary = [sum(row[d] for row in block) / len(block) for d in range(len(K[0]))]
        out.append(summary)
    return out
```

### 步骤 2: compressed-branch 注意力

运行softmax 注意力 of the 查询 against the compressed keys. The compressed-branch scores double as the 信号 for top-k selection.

### 步骤 3: top-k 块 selection

选择the indices of the `k` highest-scoring compressed 块. Load the original uncompressed 词元 from those 块 and run 注意力 on them.

### 步骤 4: sliding-window 注意力

Take the last `w` 词元 and run standard 注意力 against them.

### 步骤 5: gate + combine

一个small MLP on the 查询 produces three gate 权重. The final 输出 is a weighted sum of the three branch outputs.

### 步骤 6: 计算 counting

Print the number of keys attended per 查询 for each branch and the total. Compare to `N` (full 注意力). On a 1024-词元 synthetic with `l = 32, k = 4, w = 128`, NSA sees `32 + 128 + 128 = 288` keys per 查询 versus 1024 for full 注意力 — 3.5x fewer.

## 实际使用

NSA is shipping in DeepSeek's own long-context 预训练 流水线. Integration status in public 推理 stacks as of April 2026:

- **DeepSeek internal**: native, published 权重 use NSA or its successor DSA (Deepseek 稀疏 注意力).
- **vLLM**: experimental NSA support in development for DeepSeek-V3.x 权重.
- **SGLang**: NSA benchmarks published; 生产 path follows vLLM.
- **llama.cpp / CPU**: not supported; overhead of the kernel decomposition is not worth it at CPU throughput.

当to reach for NSA:

- Pre-training or continued-training run targeting 64k-plus 上下文 with a serious 计算 预算.
- 推理 of DeepSeek's own long-context checkpoints. The 权重 are NSA-native.

当not to:

- Serving an existing dense-attention pre-trained 模型. You cannot retrofit NSA without continued 训练.
- 上下文 under 16k. The three-branch overhead dominates the savings.
- Batch-1 interactive chat. Latency-sensitive decode benefits, but only at long contexts.

## 交付成果

这lesson produces `outputs/skill-nsa-integrator.md`. Given a long-context 预训练 run specification, it produces an NSA integration plan: 压缩 块 size, top-k, sliding window, gate MLP 宽度, kernel choice, and the specific long-context evals that would justify the 架构 change.

## 练习

1. 运行`code/main.py` on a 1024-词元 synthetic. Sweep `(l, k, w)` across three presets and print 计算 counts. Identify the preset that achieves the lowest key-count per 查询 while keeping 95% recall against full 注意力 on a needle-in-haystack test.

2. Replace the mean-pool compressor with a tiny learned MLP (2-层, 隐藏 32). 训练 it on a synthetic 任务 where the 信号 is the average of a 块. Measure the perplexity gap against the mean-pool 基线 on held-out 数据.

3. Implement the gate MLP. It takes the 查询 as 输入 and outputs three scalars. Show that the gate behaves sensibly: near-uniform weighting on random 查询, heavy 权重 on the selected branch when the 查询 hits a far-back 块.

4. 计算 the KV 缓存 内存 预算 for an NSA-enabled 70B 模型 at 128k 上下文. KV 头 are 8, 头 dim 128, BF16. Compare to full 注意力 and to MLA (Phase 10 · 14 showed MLA's numbers). Identify the 序列 length where NSA's fine-grained branch KV 缓存 equals full 注意力.

5. Read Section 4 of the NSA paper (arXiv:2502.11089) and explain in three sentences why the compressed branch's 注意力 scores are reused for top-k selection rather than computing a separate 路由 分数. Tie the 答案 to 梯度 流.

## Key Terms

|Term|What people say|What it actually means|
|------|----------------|------------------------|
|Compressed branch|"Coarse view"|注意力 over block-averaged keys that provides global 上下文 in O(N/l) keys per 查询|
|Selected branch|"Top-k 块"|Fine-grained 注意力 over the `k` 块 with highest compressed-branch scores|
|Sliding window|"Local 上下文"|注意力 over the last `W` 词元 for short-range patterns|
|Native trainability|"Pre-train with the sparsity on"|The sparsity pattern is learned during 预训练, not bolted on at 推理|
|压缩 块 size l|"Group size for coarse view"|How many 词元 get merged into one summary; 32-64 typical|
|Top-k|"块 to keep"|Number of compressed 块 whose uncompressed 词元 get read; 16 typical|
|Sliding window W|"Local 注意力 radius"|Typically 512; shorter hurts local coherence, longer wastes 计算|
|Branch gate|"How to mix the three"|Per-position MLP 输出 that 权重 the three branches' contributions|
|Hardware 对齐|"Kernel-friendly sparsity"|稀疏 pattern chosen so that the actual GPU kernel achieves the theoretical speedup|
|DSA|"NSA's successor"|Deepseek 稀疏 注意力, the 架构 that followed NSA in DeepSeek's lineage|

## 延伸阅读

- [Yuan et al. — Native Sparse Attention: Hardware-Aligned and Natively Trainable Sparse Attention (arXiv:2502.11089, ACL 2025 Best Paper)](https://arxiv.org/abs/2502.11089) — the paper
- [DeepSeek-V3 Technical Report (arXiv:2412.19437)](https://arxiv.org/abs/2412.19437) — the 架构 family NSA 目标
- [Moonshot AI — MoBA: Mixture of Block Attention for Long-Context LLMs (arXiv:2502.13189)](https://arxiv.org/abs/2502.13189) — concurrent work, MoE-style 注意力 over 块
- [Beltagy et al. — Longformer: The Long-Document Transformer (arXiv:2004.05150)](https://arxiv.org/abs/2004.05150) — sliding-window origins
- [Xiao et al. — StreamingLLM: Efficient Streaming Language Models with Attention Sinks (arXiv:2309.17453)](https://arxiv.org/abs/2309.17453) — 推理-time sparsity 基线 NSA improves on
- [Dao et al. — FlashAttention-2 (arXiv:2307.08691)](https://arxiv.org/abs/2307.08691) — the full-attention 基线 NSA kernels beat at 64k
