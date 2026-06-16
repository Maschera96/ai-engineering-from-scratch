# Jamba — Hybrid SSM-Transformer

> 状态 space 模型 (SSMs) and transformers want different things. Transformers buy 质量 via 注意力 at quadratic 成本. SSMs buy linear-time 推理 and constant 内存 via a recurrence but lag 质量. AI21's Jamba (March 2024) and Jamba 1.5 (August 2024) put them in the same 模型: 1 Transformer 层 for every 7 Mamba 层, MoE on every other 块, and a 256k 上下文 window that fits on a single 80GB GPU. Mamba-3 (ICLR 2026) tightens the SSM side with complex-valued 状态 spaces and MIMO projections. This lesson reads both architectures end to end and explains why the hybrid recipe has survived three years of 扩展 when pure-SSM and pure-Transformer long-context attempts have not.

**类型：** Learn
**语言：** Python (stdlib, layer-mix calculator)
**先修：** Phase 10 · 14 (open-model architectures), Phase 10 · 17 (native 稀疏 注意力)
**时间：** 约 60 分钟

## 学习目标

- 解释the three primitives in a Jamba 块 — Transformer 层, Mamba 层, MoE — and the 1:7:even interleaving recipe.
- 状态 what an SSM's recurrence looks like at a high level and why it enables constant-memory 推理.
- 计算 the KV 缓存 footprint of a Jamba 模型 at 256k 上下文 and compare to what a pure-Transformer 模型 would need.
- Name the three Mamba-3 innovations (exponential-trapezoidal discretization, complex-valued 状态 update, MIMO) and the problem each one 目标.

## 问题

注意力 is quadratic in 序列 length. 状态 space 模型 are linear. That difference compounds: at 256k 词元, a Transformer 注意力 map is 65B entries per 头; an SSM's recurrent 状态 is fixed-size regardless of 序列 length.

Pure-SSM 模型 (Mamba, Mamba-2) match Transformer perplexity at small scales but lag on state-tracking tasks and fail on some categories of in-context 检索. The intuition: SSMs compress history into a fixed 状态, and when history is long, information leaks. 注意力 remembers everything exactly but pays quadratic 成本.

这个obvious fix: use both. Put Transformer 层 where exact recall matters. Use SSM 层 elsewhere. Tune the 比例. Jamba is the first production-grade 模型 to ship this hybrid recipe at 规模 (52B total, 12B active, 256k 上下文, single 80GB GPU). Jamba 1.5 extends the family to 398B total / 94B active. Mamba-3 (ICLR 2026) is the current-best pure-SSM 基线 that hybrids can be rebuilt around.

这lesson reads all three papers and produces the mental 模型 for "pick the right 比例."

## 概念

### An SSM in one page

一个状态 space 模型 processes a 序列 `x_1, ..., x_N` via a fixed-size 状态 `h`:

```text
h_t = A h_{t-1} + B x_t
y_t = C h_t
```

At each 步骤 the 状态 evolves via a linear dynamics `A`, takes 输入 `B x_t`, and emits 输出 `C h_t`. `A, B, C` can be learned. Note the critical property: computing `y_t` needs only `h_{t-1}` and `x_t`, not any earlier `x`. 内存 is constant. 推理 is O(1) per 词元.

这个trick for modeling 质量 is the structure of `A`. S4 (Gu 2021) used a highly 结构化 matrix that could be evaluated efficiently as a long convolution during 训练. Mamba (Gu, Dao 2023) replaced the fixed `A, B, C` with data-dependent ones (the "selective" part). Mamba-2 (2024) further simplified the structure. Mamba-3 (2026) re-adds complexity in specific places.

这个key property: for a 解码器 LLM, an SSM 层 is a drop-in replacement for an 注意力 层, with fixed-size per-layer 状态 instead of a growing KV 缓存.

### The Jamba 块

一个Jamba 块 interleaves 层 according to two numbers:

- `l`: the attention-to-Mamba 比例. Jamba uses `l = 8`, meaning 1 Transformer 层 for every 7 Mamba 层 (7 Mamba + 1 注意力 = 8 层 per group).
- `e`: the MoE frequency. Jamba uses `e = 2`, meaning every other 层 applies MoE.

这个层 序列 within a 块:

```text
M  M  M  M  M  M  M  A    (7 Mamba + 1 Attention)
|  M  |  M  |  M  |  M    (where | marks MoE applied)
```

Each Jamba 块 is 8 层. At 4 块 deep (32 层 total), you get 28 Mamba and 4 注意力 层. 16 of those use MoE.

### Why the 1:7 比例

AI21 ran ablations: what 比例 of attention-to-Mamba gives the best perplexity-per-parameter AND in-context recall on their long-context evals?

- Too much 注意力 (1:1): 质量 goes up but 内存 and speed degrade.
- Too little 注意力 (1:15): 内存 is great but in-context 检索 fails.
- Sweet spot: 1:7 or 1:8.

这个intuition: the Transformer 层 handle exact recall and 状态 tracking. The Mamba 层 handle the cheap bulk of processing.

### Positional encoding

Mamba 层 are themselves position-aware (via the recurrence). 注意力 层 in the original Mamba-based hybrids did not use RoPE — the SSM 层 provided position info. Jamba 1.5 adds RoPE to the 注意力 层 for longer-context 泛化, a post-hoc refinement based on empirical long-context 评估.

### The 内存 预算

For a Jamba-1 shape (32 层: 28 Mamba + 4 注意力, 隐藏 4096, 32 注意力 头):

- KV 缓存 (注意力 层 only): `2 * 4 * 32 * 128 * 256k * 2 = 8.4 GB` at 256k BF16. Only the 4 注意力 层 contribute.
- SSM 状态: `28 * hidden * state_size` per 词元 prefix, but this is a fixed-size per 层, not 扩展 with 序列 length. Typical Mamba 状态 is 16 per 特征, 隐藏 4096: `28 * 4096 * 16 * 2 = 3.7 MB` total.

比较to a pure Transformer at 32 层, same 隐藏, full MHA at 32 头: `2 * 32 * 32 * 128 * 256k * 2 = 128 GB` at 256k BF16. An 8x reduction in KV 缓存. Even against the GQA(8) 基线 most 2024 模型 use (`2 * 32 * 8 * 128 * 256k * 2 = 32 GB`), Jamba's 1:7 hybrid at 16 GB is still 2x smaller.

那is what AI21 means by "256k 上下文 on a single 80GB GPU." The KV 缓存 of a full-MHA pure Transformer would not fit; even a GQA 基线 leaves no room for 权重 and activations; Jamba's does.

### Mamba-3: the pure-SSM 基线 in 2026

Mamba-3 (ICLR 2026, arXiv:2603.15569) introduces three innovations on the pure-SSM side:

1. **Exponential-trapezoidal discretization.** Replaces the Euler-method discretization in Mamba-2 with a more expressive recurrence. Convolution-like operation applied on the state-input within the core recurrence, rather than as an outer convolution on `x_t`.

2. **Complex-valued 状态 update.** Previous Mambas reduced the 状态 matrix from complex (S4) to 真实 diagonal (Mamba) to scaled 身份 (Mamba-2). Mamba-3 re-adds complex values — equivalent to a data-dependent rotary 嵌入 on the 状态. This restores state-tracking capabilities that previous real-valued simplifications 成本.

3. **Multi-input multi-output (MIMO) projections.** Instead of per-特征 scalar projections, use matrix-valued projections. Improves modeling power and 推理-time hardware utilization without increasing decode 延迟.

At 1.5B 参数, Mamba-3 improves average downstream accuracy by 0.6 points over Gated DeltaNet; the MIMO variant adds 1.2 more for a total 1.8-point gain. At the same 状态 size, Mamba-3 matches Mamba-2 with half the 状态.

Mamba-3 is not yet shipping in a 生产 hybrid at 规模 — but it is the obvious candidate for the SSM side of the next Jamba-class 模型.

### When to reach for a hybrid

Hybrids win when:

- 上下文 is long enough that pure Transformer KV 缓存 becomes painful (64k+).
- Tasks mix short-range structure (good for SSM) with long-range recall (needs Transformer).
- 你want to deploy on single-GPU 内存 budgets where the Transformer KV 缓存 alone would not fit.

Hybrids lose when:

- 上下文 is short (under 16k). The SSM overhead is wasted; pure Transformer is fine.
- Tasks need everywhere-to-everywhere 注意力 (deep 推理, multi-document cross-reference). The sparsity of 注意力 层 in the hybrid hurts.
- 你are 扩展 to trillion-parameter frontier 模型. Pure-Transformer + MLA + MoE (DeepSeek-V3 风格) is currently winning the capability race.

### The competitive landscape

|模型|Family|规模|Unique claim|
|-------|--------|------|-------------|
|Mamba-2|pure SSM|3B|linear time, constant 内存|
|Jamba|hybrid|52B/12B|256k on 80GB|
|Jamba 1.5 Large|hybrid|398B/94B|enterprise-grade long-context|
|Mamba-3|pure SSM|1.5B (paper)|state-tracking restored|
|DeepSeek-V3|pure Transformer + MoE|671B/37B|frontier capability|

这个2026 landscape: pure-Transformer MoE dominates the frontier, but hybrids own the 256k-plus 上下文 niche. Mamba-3's state-tracking wins may push hybrid ratios lower (more SSM, less 注意力) in the next 生成.

```figure
swiglu-ffn
```

## 实际使用

`code/main.py` is a 内存 calculator for hybrid architectures. Given an SSM-Transformer 比例 and a hidden-size / layer-count 配置, it computes:

- KV 缓存 at 目标 上下文.
- SSM 状态 内存.
- Total 内存 at 上下文 N for a range of 模型 shapes.

这个calculator supports:

- Pure-Transformer 基线 (KV 缓存 grows with N).
- Jamba-style 1:7 hybrid.
- Pure-SSM (no KV 缓存 at all).

这个numbers are direct from the Jamba-1 and Jamba-1.5 papers for published shapes and extrapolated for hypothetical variants.

Integration considerations for a 真实 deployment:

- Most 生产 推理 servers (vLLM, SGLang) support Jamba and Mamba. Check the specific version.
- At 256k 上下文, Jamba's 内存 advantage shows up in concurrent-request throughput. On the same VRAM you fit more Jamba sequences than Transformer sequences.
- Mamba-3 as a standalone 模型 is not yet shipping in 生产 — research preview at 1.5B.

## 交付成果

这lesson produces `outputs/skill-hybrid-picker.md`. Given a workload specification (上下文 length profile, 任务 mix, 内存 预算), it recommends between a pure Transformer, a Jamba-style hybrid, and a pure SSM, with explicit 推理 about the 内存 and 质量 取舍.

## 练习

1. 运行`code/main.py` to 计算 KV 缓存 at 256k 上下文 for a 32-层 pure Transformer (隐藏 4096, 32 头) and for a Jamba-1 hybrid of the same shape. Verify the ~8x 内存 reduction the AI21 paper claims.

2. Modify the calculator to 模型 a 1:3 hybrid (4 Mamba : 1 注意力) and a 1:15 hybrid (14 Mamba : 1 注意力). Plot KV 缓存 vs 比例. At what 比例 does the KV 缓存 equal the SSM 状态 内存?

3. Read Section 3 of the Jamba paper (arXiv:2403.19887). Explain why AI21 uses Mamba-1 rather than Mamba-2 despite Mamba-2 being faster. Hint: the hybrid ablation section 文档 this.

4. 计算 the 参数 overhead of MoE-every-other-layer in Jamba 1.5 Large (398B total, 94B active). Compare the active 比例 to DeepSeek-V3 (37B/671B) and explain why Jamba's 架构 pushes the active 比例 higher.

5. Read Section 3 of the Mamba-3 paper (arXiv:2603.15569). Explain in three sentences why a complex-valued 状态 update is equivalent to a data-dependent rotary 嵌入. Tie the 答案 to Phase 7 · Lesson 04's RoPE derivation.

## Key Terms

|Term|What people say|What it actually means|
|------|----------------|------------------------|
|状态 space 模型 (SSM)|"Recurrence with a fixed 状态"|A 层 with a learned recurrence `h_t = A h_{t-1} + B x_t`; constant 内存 per 词元|
|Selective SSM|"Mamba's trick"|Data-dependent A, B, C 参数 that give the 模型 gating-like selectivity at linear time|
|Attention-to-Mamba 比例|"How many 注意力 层"|In Jamba, `l = 8` means 1 注意力 层 per 7 Mamba 层|
|Jamba 块|"The 8-层 group"|One 注意力 + seven Mamba + MoE on alternate positions|
|SSM 状态|"The 隐藏 buffer"|修复ed-size per-layer 状态 that replaces the KV 缓存 for Mamba 层|
|256k 上下文|"Jamba's flagship number"|The 序列 length Jamba-1 fits on a single 80GB GPU; pure Transformer cannot at that size|
|Mamba-3|"2026 pure SSM"|Current-best pure-SSM 架构 with complex 状态 + MIMO; the 基线 hybrids rebuild around|
|MIMO|"Multi-input multi-output"|Mamba-3 innovation using matrix-valued projections instead of scalar per-特征|
|Exponential-trapezoidal discretization|"Mamba-3's recurrence"|More expressive recurrence that subsumes Mamba-2's Euler-method discretization|
|Hybrid 架构|"Mix 注意力 and SSM"|Any 模型 that interleaves Transformer and SSM 层; Jamba is the 生产 archetype|

## 延伸阅读

- [Lieber et al. — Jamba: A Hybrid Transformer-Mamba Language Model (arXiv:2403.19887)](https://arxiv.org/abs/2403.19887) — the original Jamba paper, 比例 ablations, 256k 上下文 claim
- [AI21 — Jamba 1.5: Hybrid Transformer-Mamba at Scale (arXiv:2408.12570)](https://arxiv.org/abs/2408.12570) — the scaled-up family, 398B/94B and 12B/52B public releases
- [Gu, Dao — Mamba: Linear-Time Sequence Modeling with Selective State Spaces (arXiv:2312.00752)](https://arxiv.org/abs/2312.00752) — the selective SSM paper Jamba builds on
- [Dao, Gu — Mamba-2 (arXiv:2405.21060)](https://arxiv.org/abs/2405.21060) — the simplified structured-state-space successor
- [Lahoti et al. — Mamba-3 (arXiv:2603.15569, ICLR 2026)](https://arxiv.org/abs/2603.15569) — complex-valued 状态, MIMO, the 2026 pure-SSM frontier
- [Gu et al. — Efficiently Modeling Long Sequences with Structured State Spaces (arXiv:2111.00396)](https://arxiv.org/abs/2111.00396) — the S4 paper, the SSM genealogy's starting point for LLMs
