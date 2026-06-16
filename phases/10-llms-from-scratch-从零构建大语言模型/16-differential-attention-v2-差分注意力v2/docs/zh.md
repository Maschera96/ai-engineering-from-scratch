# Differential 注意力 (V2)

> Softmax 注意力 spreads a small amount of 概率 over every non-matching 词元. Over 100k 词元 that 噪声 adds up and drowns the 信号. Differential Transformer (Ye et al., ICLR 2025) fixes it by computing 注意力 as the difference of two softmaxes, subtracting the shared 噪声 floor. DIFF V2 (Microsoft, January 2026) is the production-stack rewrite: 匹配 decode 延迟 to 基线 Transformer, no custom kernels, FlashAttention-compatible. This lesson is V1 to V2 end-to-end, with a working toy implementation of the difference operation you can run in stdlib Python.

**类型：** Build
**语言：** Python (stdlib)
**先修：** Phase 7 · 02 (self-attention), Phase 7 · 15 (注意力 variants), Phase 10 · 14 (架构 walkthrough)
**时间：** 约 60 分钟

## 学习目标

- 状态 precisely why softmax 注意力 has a 噪声 floor and why it grows with 上下文 length.
- Derive the differential 注意力 formula and explain why the subtraction cancels the shared 噪声 component while preserving 信号.
- Walk the V1-to-V2 diff: what got faster, what got simpler, what got more stable, and why each change was necessary for 生产 预训练.
- Implement differential 注意力 from scratch in pure Python and empirically verify the noise-cancellation property on a synthetic signal-plus-noise 查询.

## 问题

Standard softmax 注意力 has a mathematical property that turns into an operational headache at 规模. For a 查询 `q`, the 注意力 权重 are `softmax(qK^T / sqrt(d))`. Softmax can never produce exact zeros — every non-matching 词元 gets some positive mass. That residual mass is 噪声, and it scales with 上下文 length. At 128k 词元, even if each non-matching 词元 gets only 0.001% of the 概率, 127,999 of them combined contribute about 12% of the total. The 模型 has to learn to route around a 噪声 floor that grows with 上下文.

Empirically this shows up as attention-head interference: hallucinated citations in long-context RAG, lost-in-the-middle failures on 100k-词元 检索 tasks, and subtle accuracy degradation on needle-in-haystack benchmarks past 32k. The Differential Transformer paper (arXiv:2410.05258, ICLR 2025) measured the gap: DIFF Transformers hit lower perplexity, higher long-context accuracy, and fewer hallucinations than same-size baselines.

DIFF V1 had three problems that kept it out of frontier 预训练 pipelines. Its value 缓存 had to be loaded twice per decode 步骤, it required custom CUDA kernels that broke FlashAttention compatibility, and its per-head RMSNorm destabilized long-run 训练 at 70B-plus 规模. DIFF V2 (Microsoft unilm blog, January 20, 2026) fixed all three. This lesson walks both versions, builds the difference operator, and benchmarks 噪声 cancellation on a toy 查询.

## 概念

### The 噪声 floor of softmax

For a 查询 `q` and keys `K = [k_1, ..., k_N]`, 注意力 权重 are:

```text
w_i = exp(q . k_i / sqrt(d)) / sum_j exp(q . k_j / sqrt(d))
```

No `w_i` is ever zero. If `k_i` is completely unrelated to `q`, the 分数 `q . k_i` is not 0 — it fluctuates around zero with 方差 `||q||^2 / d`. After softmax 归一化, each unrelated 词元 still contributes `O(1/N)` to the weighted sum. The total contribution of unrelated 词元 is `O((N-1)/N) = O(1)` — not a small quantity.

What the 模型 wants is something like a hard top-k: high 权重 on 匹配 词元, near-zero 权重 everywhere else. Softmax is too smooth to do that directly.

### The differential idea

Split each 头's Q and K projections into two: Q = (Q_1, Q_2) and K = (K_1, K_2). 计算 two 注意力 maps:

```text
A_1 = softmax(Q_1 K_1^T / sqrt(d))
A_2 = softmax(Q_2 K_2^T / sqrt(d))
```

输出：

```text
DiffAttn = (A_1 - lambda * A_2) V
```

这个subtraction cancels whatever 噪声 分布 the two maps share. If both maps have roughly uniform 权重 on the 127k unrelated 词元 (which they will, at random initialization), those cancel. The 信号 — peaked 权重 on the few actually relevant 词元 — only cancels if it appears in both maps at the same magnitude, which it will not once the 模型 trains.

`lambda` is a learnable scalar per 头, parameterized as `lambda = exp(lambda_q1 dot lambda_k1) - exp(lambda_q2 dot lambda_k2) + lambda_init`. It can be negative. `lambda_init` defaults to a small positive number like 0.8.

### Why this matches headed noise-canceling

Think of two noisy microphones recording the same voice. Both pick up the speaker plus correlated background 噪声. Subtract one from the other and the shared 噪声 drops out. The voice survives because the two signals differ in phase or amplitude by enough to prevent full cancellation. The per-head `lambda` learns exactly this balance.

### V1 vs V2: the diff

V1 kept the 参数 count equal to the 基线 Transformer. To get two 查询 per 头 it halved the 头 维度. That 成本 头 expressiveness and — more painfully — halved the value 缓存 per 头. Decode had to load the value 缓存 twice per 步骤 (once per softmax branch). Result: decode slower than 基线 despite 匹配 参数 count.

V2 doubles the number of 查询 头 and keeps the KV 头 the same (borrowing 参数 from the up-projection). The 头 维度 stays the same as 基线. After the subtraction, the extra 维度 is projected back down to match 基线 Transformer's O_W projection. Three things happen at once:

1. Decode speed matches 基线 (KV 缓存 is loaded once).
2. FlashAttention runs unchanged (no custom kernel).
3. Arithmetic intensity at decode goes up (more 计算 per byte loaded from HBM).

V2 also removes the per-head RMSNorm that V1 used to stabilize the subtraction. At 70B-class 预训练 scales, that RMSNorm destabilized late 训练. V2 replaces it with a simpler initialization scheme that keeps 训练 stable without the extra module.

### When to reach for it

|Workload|Benefit|
|----------|---------|
|Long-context RAG (64k+)|Cleaner 注意力 maps, fewer hallucinated citations|
|Needle-in-haystack benchmarks|Substantial accuracy lift past 32k|
|Multi-document QA|Less cross-document interference|
|Code completion at 8k|Marginal, not worth the 架构 change|
|Short chat (< 4k)|Essentially indistinguishable from 基线|

这个value grows with 上下文 length. At 4k 词元 the 噪声 floor is small enough that standard 注意力 is fine. At 128k it is hurting you.

### How it stacks with other 2026 knobs

|特征|Compatible with DIFF V2?|
|---------|------------------------|
|GQA|Yes (V2 increases Q 头, not KV 头)|
|MLA (DeepSeek)|Yes in principle, no published paper combining them|
|MoE|Yes (注意力 is independent of MLP 块)|
|RoPE|Yes (unchanged)|
|YaRN / long-context 扩展|Yes (exactly where DIFF helps most)|
|FlashAttention|Yes in V2 (was no in V1)|
|Speculative decoding|Yes (注意力 change is invisible to the spec-decode 循环)|

```figure
differential-attention
```

## 动手构建

`code/main.py` implements differential 注意力 in pure Python. A toy 查询 with known signal-plus-noise structure lets you measure the noise-cancellation 比例 directly.

### 步骤 1: standard softmax 注意力

Stdlib matrix ops: lists of lists, manual matmul, softmax with numerical-stability subtraction of the max.

```python
def softmax(row):
    m = max(row)
    exps = [math.exp(x - m) for x in row]
    s = sum(exps)
    return [e / s for e in exps]
```

### 步骤 2: split Q, K into two halves

V1 风格: halve the 头 维度. V2 风格: keep the 头 维度 and double the number of 头. The toy implementation uses V1 for pedagogical clarity — the math is identical, only the bookkeeping differs.

### 步骤 3: two softmax branches + subtraction

```python
A1 = [softmax([dot(q1, k) / scale for k in K1]) for q1 in Q1]
A2 = [softmax([dot(q2, k) / scale for k in K2]) for q2 in Q2]
diff_weights = [[a1 - lam * a2 for a1, a2 in zip(r1, r2)] for r1, r2 in zip(A1, A2)]
out = [[sum(w * v[j] for w, v in zip(row, V)) for j in range(d_v)] for row in diff_weights]
```

Note: the 输出 权重 can be negative. That is fine — the value 缓存 still handles signed contributions. The subsequent V projection absorbs the sign.

### 步骤 4: 噪声 cancellation measurement

构建a synthetic 序列 of length 1024. Place the 信号 词元 at a known position, fill the rest with 噪声. 计算 (a) standard softmax 注意力 权重 on the 信号 position and (b) differential 注意力 权重. Measure the 比例 of signal-to-noise in each. DIFF 注意力 reliably produces a higher signal-to-noise 比例 by a factor of 3x-10x depending on how much the two branches have been 训练后的 to differ.

### 步骤 5: V1 vs V2 参数 accounting

给定a 配置 (隐藏=4096, 头=32, d_head=128), print:

- 基线 Transformer: Q, K, V each size `hidden * hidden`, MLP at 4 * 隐藏.
- DIFF V1: Q, K each size `hidden * hidden`, V size `hidden * hidden` (unchanged), 头 dim halved internally. Adds per-head `lambda` 参数 (O(heads * d_head)).
- DIFF V2: Q size `2 * hidden * hidden`, K size `hidden * hidden`, V size `hidden * hidden`. Extra dim projected back down before O_W. Adds same `lambda` 参数.

这个toy measures the extra 参数 成本 for V2 (roughly `hidden * hidden` extra per 注意力 块) and prints it.

## 实际使用

DIFF V2 is not yet shipping in every 生产 推理 服务器 as of April 2026, but integration is underway in vLLM and SGLang. Meanwhile the pattern shows up in:

- Microsoft internal long-context 生产 模型.
- Research replications in several 开放 模型 训练 runs targeting 256k-plus 上下文.
- Hybrid architectures that combine DIFF 注意力 with sliding-window 注意力 on alternate 层.

当you would reach for this in 2026:

- 训练 a new 模型 from scratch targeting 64k-plus effective 上下文. Add differential 注意力 from the start; retraining later is expensive.
- Fine-tuning a long-context 模型 where lost-in-the-middle failures dominate your 评估. A LoRA on the Q projections can approximate the DIFF structure.

当you would not:

- 你are serving a pre-trained 稠密 模型 with stable long-context performance. The retraining 成本 rarely pays back on existing 权重.
- 你的上下文 is always under 16k. 噪声 floor is negligible.

## 交付成果

这lesson produces `outputs/skill-diff-attention-integrator.md`. Given a 模型 架构, 目标 上下文 length, 幻觉 profile, and 训练 预算, it produces an integration plan for adding differential 注意力 to a new 预训练 run or LoRA fine-tune.

## 练习

1. 运行`code/main.py`. Verify the signal-to-noise 比例 reported for differential 注意力 is higher than standard softmax 注意力 on the synthetic 查询. Vary the 噪声 amplitude and show the crossover point where standard 注意力 becomes unusable.

2. 计算 the parameter-count delta from 基线 to DIFF V1 and from 基线 to DIFF V2 for a 7B-class 模型 (隐藏=4096, 头=32, d_head=128, 32 层). Show which components gained 参数 and which stayed the same.

3. Read Section 3 of the DIFF V1 paper (arXiv:2410.05258) and Section 2 of the DIFF V2 Hugging Face blog. In two sentences, explain why the V1 per-head RMSNorm was necessary and why V2 could remove it without causing 训练 divergence.

4. Implement an ablation: 计算 differential 注意力 with `lambda = 0` (pure first softmax) and `lambda = 1` (full subtraction). On the synthetic 查询, measure how signal-to-noise changes across the sweep. Identify the `lambda` that maximizes signal-to-noise.

5. Extend the toy to GQA + DIFF V2. Pick 8 KV 头 and 32 Q 头. Show that the KV 缓存 size matches a 基线 GQA 模型 with the same (8, 32) configuration.

## Key Terms

|Term|What people say|What it actually means|
|------|----------------|------------------------|
|Differential 注意力|"Two softmaxes minus each other"|Split Q, K into two halves, 计算 two softmax maps, subtract the second (scaled by lambda) from the first, then multiply by V|
|噪声 floor|"The non-zero tail of softmax"|The O(1/N) 权重 softmax puts on every unrelated 词元, which sums to O(1) across long contexts|
|lambda|"The subtraction 规模"|Per-head learnable scalar parameterized as `exp(lq1.lk1) - exp(lq2.lk2) + lambda_init`; can be negative|
|DIFF V1|"The ICLR 2025 version"|Original Differential Transformer; halves 头 dim to preserve 参数 count, needs custom kernel, slower decode|
|DIFF V2|"The January 2026 fix"|Doubles Q 头 keeping KV 头; matches 基线 decode speed and works with FlashAttention|
|Per-head RMSNorm|"The V1 stabilizer"|Extra norm V1 applied after the difference; V2 removed it to prevent late-training instability|
|Signal-to-noise 比例|"How much 注意力 is wasted"|比例 of 权重 on the true 信号 position to average 权重 on unrelated positions|
|Lost in the middle|"Long-context failure mode"|Empirical phenomenon where 检索 accuracy dips for 文档 in the middle of a long 上下文 — DIFF 注意力 reduces this|
|Arithmetic intensity|"FLOPs per byte loaded"|比例 V2 increased at decode by doubling 查询 per KV load; important for memory-bound decode|

## 延伸阅读

- [Ye et al. — Differential Transformer (arXiv:2410.05258, ICLR 2025)](https://arxiv.org/abs/2410.05258) — the original paper with noise-cancellation theory and long-context ablations
- [Microsoft unilm — Differential Transformer V2 (Hugging Face blog, January 2026)](https://huggingface.co/blog/microsoft/diff-attn-v2) — the production-stack rewrite, 匹配 基线 decode, FlashAttention-compatible
- [Understanding Differential Transformer Unchains Pretrained Self-Attentions (arXiv:2505.16333)](https://arxiv.org/abs/2505.16333) — theoretical analysis of why the subtraction recovers 预训练 注意力 structure
- [Shared DIFF Transformer (arXiv:2501.17900)](https://arxiv.org/html/2501.17900) — parameter-sharing variant
- [Vaswani et al. — Attention Is All You Need (arXiv:1706.03762)](https://arxiv.org/abs/1706.03762) — the 基线 Transformer DIFF subtracts from
- [Liu et al. — Lost in the Middle (arXiv:2307.03172)](https://arxiv.org/abs/2307.03172) — the long-context 基准 DIFF 注意力 目标
