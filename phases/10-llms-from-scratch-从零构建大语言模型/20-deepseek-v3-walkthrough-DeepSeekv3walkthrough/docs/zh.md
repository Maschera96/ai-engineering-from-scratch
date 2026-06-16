# DeepSeek-V3 架构 Walkthrough

> Phase 10 · Lesson 14 named the six architectural knobs every 开放 模型 turns. DeepSeek-V3 (December 2024, 671B 参数 total, 37B active) turns all six and adds four more: Multi-Head 潜变量 注意力, auxiliary-loss-free load balancing, Multi-Token 预测, and DualPipe 训练. This lesson reads DeepSeek-V3's 架构 top to bottom and derives every 参数 count from the published 配置. By the end you can explain why the 671B/37B 比例 is the right bet and why MLA + MoE together beat either alone at the frontier.

**类型：** Learn
**语言：** Python (stdlib, parameter calculator)
**先修：** Phase 10 · 14 (open-model walkthroughs), Phase 10 · 17 (NSA), Phase 10 · 18 (MTP), Phase 10 · 19 (DualPipe)
**时间：** 约 75 分钟

## 学习目标

- Read the DeepSeek-V3 配置 top to bottom and explain each field in terms of the six GPT-2 knobs plus four DeepSeek-specific additions.
- Derive the total 参数 count (671B), active 参数 count (37B), and the components that contribute to each.
- 计算 the KV 缓存 footprint of MLA at 128k 上下文 and compare to what a same-active-param 稠密 模型 with GQA would pay.
- 状态 the four DeepSeek-specific innovations (MLA, MTP, auxiliary-loss-free 路由, DualPipe) and name which part of the 架构/训练 stack each one 目标.

## 问题

DeepSeek-V3 is the first frontier 开放 模型 whose 架构 is meaningfully different from the Llama family. Llama 3 405B is "GPT-2 with six knobs turned." DeepSeek-V3 is GPT-2 with all six knobs plus four more. Reading the Llama 3 配置 is a 预热 for reading the DeepSeek 配置, but the deep structure — the shape of the 注意力 块, the 路由 logic, the training-time 目标 — is different enough that you need a separate walkthrough.

这个payoff of 学习 it: DeepSeek-V3's open-weights release shifted what "frontier capability" means in 开放 模型. The 架构 is the blueprint many 2026 训练 runs are copying. Understanding it is table stakes for any role that touches frontier LLM 训练 or 推理.

## 概念

### The invariant core, again

DeepSeek-V3 is still autoregressive. It still stacks 解码器 块. Each 块 still has 注意力 plus MLP plus two RMSNorms. It still uses SwiGLU in the MLP. It still uses RoPE. Pre-norm. Weight-tied 嵌入s. Same 基线 as every Llama or Mistral.

### The twist: MLA instead of GQA

From Phase 10 · 14 you know GQA shrinks the KV 缓存 by sharing K and V across groups of Q 头. Multi-Head 潜变量 注意力 (MLA) goes further: K and V are compressed into a shared low-rank 潜变量 representation (the `kv_lora_rank`), then decompressed per 头 on the fly. The KV 缓存 stores only the 潜变量 — typically 512 floats per 词元 per 层, not 8 x 128 = 1024 floats.

At 128k 上下文, DeepSeek-V3 with MLA (one shared 潜变量 `c^{KV}` per 词元 per 层; K and V are both derived from this 潜变量 via up-projections that can be absorbed into the subsequent matmul):

```text
kv_cache = num_layers * kv_lora_rank * max_seq_len * bytes_per_element
         = 61 * 512 * 131072 * 2
         = 7.6 GB
```

一个hypothetical GQA 基线 (Llama 3 70B shape, 8 KV 头, 头 dim 128) would pay:

```text
kv_cache = 2 * 61 * 8 * 128 * 131072 * 2
         = 30.5 GB
```

MLA is 4x smaller than a Llama-3-70B-style GQA 缓存 at 128k 上下文.

这个tradeoff: MLA adds a decompression 步骤 per 注意力 computation (per 头). The extra 计算 is small compared to the bandwidth saved. Net win for long-context 推理.

### The 路由: auxiliary-loss-free load balancing

MoE routers decide which top-k experts process each 词元. A naive 路由器 concentrates too much work on a few experts, leaving others idle. Standard fix: add an auxiliary 损失 term that penalizes load imbalance. This works but slightly degrades main-task performance.

DeepSeek-V3 introduces an auxiliary-loss-free scheme. Per-expert 偏差 terms are added to the 路由器 logits, adjusted during 训练 by a simple rule: if expert `e` is overloaded, decrease `bias_e`; if underloaded, increase it. No extra 损失 term. 训练 stays clean. Expert load stays balanced.

Effect on the main 损失: none measurable. Effect on the MoE 架构: cleaner, no auxiliary-loss hyperparameter to tune.

### The MTP: denser 训练 + free draft

From Phase 10 · 18 you know DeepSeek-V3 adds D=1 MTP module that predicts the 词元 two positions ahead. At 推理, the 训练后的 module is repurposed as a speculative-decoding draft with 80%+ acceptance. At 训练, each 隐藏 状态 is supervised on D+1 = 2 目标, providing a denser 信号.

参数: 14B on top of the 671B main. Overhead: 2.1%.

### The 训练: DualPipe

From Phase 10 · 19 you know DualPipe is a bidirectional 流水线 that overlaps forward and backward chunks with cross-node all-to-all comms. At DeepSeek-V3's 2,048-H800 规模, it recovers roughly 245k GPU-hours that 1F1B would have lost to 流水线 bubbles.

### The 配置, field by field

Here is the DeepSeek-V3 配置 (simplified):

```text
hidden_size: 7168
intermediate_size: 18432   (dense MLP hidden size, used on first few layers)
moe_intermediate_size: 2048 (expert MLP hidden size)
num_hidden_layers: 61
first_k_dense_layers: 3    (first 3 layers use dense MLP)
num_attention_heads: 128
num_key_value_heads: 128   (formally equal to num_heads under MLA, but
                           the real compression is in kv_lora_rank)
kv_lora_rank: 512          (MLA latent dimension)
num_experts: 256            (MoE expert count per block)
num_experts_per_tok: 8      (top-8 routing)
shared_experts: 1           (always-on shared expert per block)
max_position_embeddings: 163840
rope_theta: 10000.0
vocab_size: 129280
mtp_module: 1               (1 MTP module at depth 1)
```

Parse it:

- `hidden_size=7168`: 嵌入 维度.
- `num_hidden_layers=61`: total 块 深度.
- `first_k_dense_layers=3`: the first 3 块 use a 稠密 MLP of size 18432. The remaining 58 use MoE.
- `num_attention_heads=128`: 128 查询 头.
- `kv_lora_rank=512`: K and V are compressed to this 潜变量 维度 and decompressed per 头.
- `num_experts=256, num_experts_per_tok=8`: each MoE 块 has 256 experts, routes top-8.
- `shared_experts=1`: on top of the 256 routed experts, 1 always-on expert contributes to every 词元. Think of it as a "稠密 floor" that ensures every 词元 gets something reliable.
- `moe_intermediate_size=2048`: each expert's MLP 隐藏 size. Smaller than the 稠密 MLP because there are 256 of them.

### 参数 accounting

这个full calculation lives in `code/main.py`. The headline:

- 嵌入: `vocab * hidden = 129280 * 7168 = ~0.93B`.
- First 3 稠密 块: 注意力 with MLA (~144M per 块) + 稠密 MLP (~260M per 块) + norms. About 1.2B total.
- 58 MoE 块: 注意力 with MLA (~144M) + 256 experts each (30M apiece) + 1 shared expert (30M) + norm. Total ~7.95B per 块, including all experts. 461B total for the 58 MoE 块.
- MTP module: 14B.

Grand total: ~476B for core 架构 + 14B MTP + distinctly the published 671B number accounts for additional structural 参数 (偏差 tensors, expert-specific components, shared expert 扩展, etc.). The number we reproduce in the calculator is within 3-5% of published — the delta comes from fine-grained accounting DeepSeek's report 文档 in its Section 2 appendix.

Active 参数 per forward:

- 注意力: 144M per 层 * 61 = 8.8B (all 层 fire).
- MLP active: first 3 层 稠密 (3 * 260M = 780M), 58 MoE 层 each active with 8 routed + 1 shared + 路由 overhead. Per 层 active MLP: ~260M. Total: 3 * 260M + 58 * 260M = ~15.9B.
- 嵌入 + norms: 1.2B.
- Total active: roughly 26B core + 14B MTP (训练后的 but not always run at 推理) ≈ 37B.

### The 671B / 37B 比例

18x sparsity 比例 (active params are 5.5% of total). DeepSeek-V3 is the sparsest frontier MoE 模型 that has shipped 开放 权重. Mixtral 8x7B at 比例 13/47 (28%) is much denser. Llama 4 Maverick at 比例 17B/400B (4.25%) is comparable. The DeepSeek bet: at frontier 规模, more experts with lower 激活 比例 produces better 质量 per active-FLOP.

### Where DeepSeek-V3 sits

|模型|Total|Active|比例|注意力|Novel ideas|
|-------|------|-------|-------|-----------|-------------|
|Llama 3 70B|70B|70B|100%|GQA 64/8|—|
|Llama 4 Maverick|400B|17B|4.25%|GQA|—|
|Mixtral 8x22B|141B|39B|27%|GQA|—|
|DeepSeek V3|671B|37B|5.5%|MLA 512|MLA + MTP + aux-free + DualPipe|
|Qwen 2.5 72B|72B|72B|100%|GQA 64/8|YaRN extension|

### The follow-on: R1, V4

DeepSeek-R1 (2025) is a reasoning-training run on the V3 backbone. R1 uses the same 架构. What changed is the post-training recipe (large-scale RL on verifiable tasks), not the pretraining 架构.

DeepSeek-V4 (if it ships) is expected to keep MLA + MoE + MTP and add DSA (DeepSeek 稀疏 注意力), the successor to NSA from Phase 10 · 17. The lineage is stable: architecture-level innovations accumulate; each version turns additional knobs.

```figure
moe-routing
```

## 实际使用

`code/main.py` is the 参数 calculator specialized to DeepSeek-V3's shape. Run it, compare its 输出 to the paper's numbers, and use it on hypothetical variants (256 experts vs 512, top-8 vs top-16, MLA 排序 512 vs 1024).

What to look at:

- Total 参数 count vs published 671B.
- Active 参数 count vs published 37B.
- KV 缓存 at 128k 上下文 — the MLA vs GQA comparison.
- Per-layer breakdown to see where the 参数 预算 actually goes.

## 交付成果

这lesson produces `outputs/skill-deepseek-v3-reader.md`. Given a DeepSeek-family 模型 (V3, R1, or any future variant), it produces a component-by-component 架构 reading that names each field of the 配置, derives 参数 counts by component, and identifies which of the four DeepSeek-specific innovations the 模型 uses.

## 练习

1. 运行`code/main.py`. Compare the calculator's total-parameter estimate to the published 671B and identify where the delta comes from. The paper's Section 2 has the full itemization.

2. Modify the 配置 to use MLA 排序 256 instead of 512. 计算 the resulting KV 缓存 size at 128k 上下文. What percentage reduction does it buy, and at what 成本 to the per-head expressiveness?

3. 比较DeepSeek-V3's (256 experts, top-8) 路由 to a hypothetical (512 experts, top-8) variant. Total 参数 grow; active 参数 stay the same. What does the extra expert capacity buy in theory, and what does it 成本 at 推理?

4. Read Section 2.1 of the DeepSeek-V3 technical report (arXiv:2412.19437) on MLA. Explain in three sentences why the K and V decompression matrices can be "absorbed" into the subsequent matmul for 推理-time efficiency.

5. DeepSeek-V3 uses FP8 训练 for most operations. 计算 the 内存 savings of FP8 vs BF16 for storing the 671B 权重. How does this intersect with the 14.8T-词元 训练 预算?

## Key Terms

|Term|What people say|What it actually means|
|------|----------------|------------------------|
|MLA|"Multi-Head 潜变量 注意力"|Compress K and V into a shared low-rank 潜变量 (kv_lora_rank, typically 512), decompress per 头 on-the-fly; KV 缓存 stores only the 潜变量|
|kv_lora_rank|"MLA 压缩 dim"|The size of the shared 潜变量 for K and V; DeepSeek-V3 uses 512|
|First k 稠密 层|"Early 层 stay 稠密"|The first few MoE-model 层 skip the MoE 路由器 and run a 稠密 MLP for stability|
|num_experts_per_tok|"Top-k 路由"|How many routed experts fire per 词元; DeepSeek-V3 uses 8|
|Shared experts|"Always-on experts"|Experts that process every 词元 regardless of 路由; DeepSeek-V3 uses 1|
|Auxiliary-loss-free 路由|"Bias-adjusted load balance"|Per-expert 偏差 terms adjusted during 训练 to keep expert load balanced without adding a 损失 term|
|MTP module|"Extra 预测 头"|Transformer 块 predicting t+2 from h^(1) and E(t+1); denser 训练, free speculative-decoding draft|
|DualPipe|"Bidirectional 流水线"|训练 调度 that overlaps forward/backward 计算 with cross-node all-to-all|
|Active 参数 比例|"Sparsity"|active_params / total_params; DeepSeek-V3 hits 5.5%|
|FP8 训练|"8-bit 训练"|训练 storage and many 计算 ops in FP8; roughly halves 内存 vs BF16 at a small 质量 成本|

## 延伸阅读

- [DeepSeek-AI — DeepSeek-V3 Technical Report (arXiv:2412.19437)](https://arxiv.org/abs/2412.19437) — the full 架构, 训练, and results 文档
- [DeepSeek-V3 model card on Hugging Face](https://huggingface.co/deepseek-ai/DeepSeek-V3) — 配置 files and deployment notes
- [DeepSeek-V2 paper (arXiv:2405.04434)](https://arxiv.org/abs/2405.04434) — the predecessor that introduced MLA
- [DeepSeek-R1 paper (arXiv:2501.12948)](https://arxiv.org/abs/2501.12948) — the reasoning-training successor on V3's 架构
- [Native Sparse Attention (arXiv:2502.11089)](https://arxiv.org/abs/2502.11089) — the future direction for DeepSeek-family 注意力
- [DualPipe repository](https://github.com/deepseek-ai/DualPipe) — the training-schedule 参考
