# Multi-Token 预测 (MTP)

> 每个autoregressive LLM from GPT-2 to Llama 3 trains on one 损失 per position: 预测 the next 词元. DeepSeek-V3 added a second 损失 per position: 预测 the 词元 after that. The extra 14B of 参数 (on a 671B 模型) got distilled back into the main 模型 through 梯度 流, and the 训练后的 MTP 头 were repurposed at 推理 as speculative-decoding drafters with 80%+ acceptance. 1.8× 生成 throughput came for free. This lesson builds the sequential MTP module from the DeepSeek technical report, computes the 损失 and the shared-head 参数 layout, and explains why MTP keeps the 因果 链 while Gloeckle et al.'s original 并行 MTP broke it.

**类型：** Build
**语言：** Python (stdlib)
**先修：** Phase 10 · 04 (预训练 a mini GPT), Phase 10 · 15 (推测解码)
**时间：** 约 60 分钟

## 学习目标

- 状态 the MTP 训练 目标 and derive the joint 损失 across 预测 depths.
- 解释the difference between Gloeckle et al.'s 并行 MTP 头 (2024) and DeepSeek-V3's sequential MTP modules and why the sequential design preserves the 因果 链.
- 计算 the 参数 and 内存 overhead of adding MTP modules to a 预训练 run.
- Implement one MTP module from scratch: the shared 嵌入, the per-depth transformer 块, the projection, and the shared 输出 头.

## 问题

Next-token 预测 is the standard LLM 训练 目标. Every 隐藏 状态 is supervised to 预测 exactly one thing: the immediately following 词元. That is a surprisingly weak 信号. Most of the information in a 序列 extends beyond one 词元 — structure, coherence, factuality, arithmetic 流. The 模型 has to learn those by accumulating many one-token signals over trillions of 词元.

MTP asks: what if every 隐藏 状态 were supervised to 预测 multiple future 词元 at once? Gloeckle et al. (Meta, 2024) showed this helps. Their implementation put several independent 输出 头 on top of the backbone, each predicting a different offset. 并行, simple, but the 头 saw the same 隐藏 状态 without any hierarchical refinement — and the predictions did not 链 causally, so they could not be used for 推测解码.

DeepSeek-V3 (December 2024) re-designed MTP as sequential modules that keep the 因果 链 at each 预测 深度. The 模型 predicts `t+1` from `h_i^(0)`, then predicts `t+2` from a new 隐藏 状态 `h_i^(1)` that combined `h_i^(0)` with the `E(t+1)` 嵌入, and so on. Each 深度 is its own small transformer 块. The shared 嵌入 and shared 输出 头 keep 参数 overhead modest. At DeepSeek-V3's 规模, 14B extra 参数 across MTP modules on top of 671B main-model 权重. That 2% overhead bought denser 训练 signals AND a ready-made speculative-decoding draft at 推理.

这lesson builds a single MTP module and the D-depth 损失 from scratch. The math is tidy. The implementation is 150 lines.

## 概念

### The sequential MTP recipe

DeepSeek-V3 adds `D` MTP modules on top of the main 模型. Each module `k` (for `k = 1..D`) predicts the 词元 at 深度 `k` — that is, `t_{i+k}` given a prefix through position `i`.

Module `k` consists of:

- 一个transformer 块 `T_k` with its own 注意力 and MLP.
- 一个projection matrix `M_k` that combines the previous-depth 隐藏 状态 with the 嵌入 of the next-depth ground-truth 词元.
- 这个shared 嵌入 `E` (same as the main 模型).
- 这个shared 输出 头 `Out` (same as the main 模型).

At 训练, for a prefix through position `i`, the per-depth 隐藏 状态 is:

```text
h_i^(0) = main model backbone at position i
h_i^(k) = T_k( M_k * concat(RMSNorm(h_i^(k-1)), RMSNorm(E(t_{i+k}))) )   for k >= 1
```

这个per-depth 预测 is:

```text
logits_{i+k} = Out(h_i^(k-1))   for k = 1..D
```

这个per-depth 损失 is cross-entropy against the ground-truth `t_{i+k}`:

```text
L_k = CE(logits_{i+k}, t_{i+k})
```

这个joint 损失 across depths:

```text
L_MTP = (lambda / D) * sum_{k=1..D} L_k
```

`lambda` is a small weighting factor — DeepSeek-V3 uses 0.3 for the first 10% of 训练 and 0.1 afterward. The total 训练 损失 is `L_main + L_MTP`.

### Why sequential, not 并行

Gloeckle's original 并行 MTP had D 输出 头, each directly applied to `h_i^(0)`. Each 头 predicts `t_{i+k}` from the same backbone 隐藏 状态. That trains fine, but the predictions are not conditioned on each other. You cannot use `head_1`'s 输出 to help `head_2` — the 头 fire in 并行.

DeepSeek-V3's sequential design builds `h_i^(k)` from `h_i^(k-1)` plus the actual next-token 嵌入 `E(t_{i+k})`. That preserves the 因果 链: to 预测 `t_{i+k+1}`, the module at 深度 `k+1` sees what was at `t_{i+k}`. This is structurally identical to how an autoregressive 解码器 consumes its own 输出 — making the MTP modules directly usable as speculative-decoding drafters.

At 推理: feed `h_i^(k-1)` and the drafted `t_{i+k}` into module `k+1`, get a 预测 for `t_{i+k+1}`. Repeat. That is exactly an EAGLE-style draft, using the 训练后的 MTP module as the draft network. DeepSeek-V3 reports 80%+ acceptance on the first MTP module and ~1.8× speedup.

### 参数 accounting

For a 模型 with 隐藏 `h` and 词表 `V`:

- Main 模型: billions of 参数, plus one 输出 头 of size `V * h`.
- Shared 输出 头: reuse the main 模型's 头. No extra params.
- Shared 嵌入: reuse the main 模型's 嵌入. No extra params.
- Per-MTP module:
  - Projection `M_k`: `(2h) * h = 2h^2`.
  - Transformer 块 `T_k`: 注意力 (`4h^2` for MHA) plus MLP (typically `8h^2` for SwiGLU with 比例 8/3). About `12h^2` per 块.

Total extra per module: `~14h^2`. For DeepSeek-V3's `h = 7168`, D = 1 module: `~14 * 7168^2 = ~720M` 参数 on paper. DeepSeek-V3 reports 14B — the difference is mostly expert 层 being MoE in the MTP module too.

### The speculative-decoding payoff

During 预训练, the MTP modules slow 训练 by about 10% (more forward 计算, extra 损失). The payoff is two-fold:

1. Denser 训练 信号. Each 隐藏 状态 sees D+1 supervision 目标. Measured effect on MMLU, GSM8K, MATH, HumanEval: consistent few-percentage-point improvements in DeepSeek-V3's ablations.

2. Free 推测解码 draft at 推理. The MTP module is already 训练后的 to 预测 the next few 词元. Repurposed as a draft network, it delivers 80%+ acceptance rates. At that level, N=3 or N=5 spec decoding gives 1.8× throughput. The 10% training-time 成本 pays back the first time you run 推理.

### Relation to EAGLE

EAGLE trains a small draft 模型 SEPARATELY after 预训练. MTP bakes the draft into 预训练. The two approaches converge on similar accept rates but via different pipelines:

|维度|EAGLE-3|MTP (DeepSeek-V3)|
|-----------|---------|------------------|
|When 训练后的|Post-预训练|During 预训练|
|Backward-compatible with existing 权重|Yes|No (need to re-train)|
|Draft params|1-2 transformer 层|1 transformer 块 + projection|
|Acceptance 速率|0.88-0.92|0.80+ at 深度 1|
|Benefit beyond speedup|Speculative decoding only|Denser 训练 信号 + speedup|

## 动手构建

`code/main.py` builds a single MTP module end to end: shared 嵌入, projection, transformer 块, shared 输出 头. It then computes the per-depth cross-entropy 损失 on a short synthetic 序列 and prints the 参数 count by component. A toy 词表 of 32 词元 keeps the numbers readable.

### 步骤 1: shared 嵌入 table

一个single `vocab_size x hidden` table is used by the main 模型 AND by every MTP module at every 深度. Not a second copy — literally the same tensor.

### 步骤 2: the per-depth combination

```python
def combine(prev_hidden, next_token_embed, M_k):
    # concat along feature dim, then project down to hidden
    concat = rms_norm(prev_hidden) + rms_norm(next_token_embed)  # vector addition stand-in
    projected = matvec(M_k, concat)
    return projected
```

真实 DeepSeek-V3 concatenates the two RMSNormed vectors to `[2h]` and projects with an `h x 2h` matrix. The toy uses vector addition for stdlib brevity.

### 步骤 3: the transformer 块 at 深度 k

Self-attention plus MLP. In the toy, a one-layer linear 注意力 块 and a SwiGLU MLP keep the structure visible without numpy.

### 步骤 4: the shared 输出 头

Reuse the main 模型's 输出 projection. logits over the 词表.

### 步骤 5: per-depth 损失

Cross-entropy of softmax(logits) against the ground-truth 词元 at offset `k`. Aggregate across depths with the `lambda / D` 扩展 factor.

### 步骤 6: 参数 accounting

Print the total 参数 count, the shared (嵌入, 头) count, and the per-module extra count. Show the 比例 of MTP extra to main-model size.

## 实际使用

MTP is integrated into DeepSeek-V3 (December 2024) and the DeepSeek-R1 series. At 推理:

- DeepSeek's own serving stack consumes MTP modules as speculative decoders out of the box.
- vLLM and SGLang have integration paths for DeepSeek-V3 MTP as of April 2026.
- AMD's ROCm SGLang tutorial shows a specific MTP speculative-decoding 配置 with measured 1.8× speedup on the V3 checkpoint.

当to use MTP in a new 预训练 run:

- 你control the full 预训练 流水线 and want to bank denser 训练 信号.
- 你know you will serve the 模型 at 规模 and want 推测解码 for free.
- 你的隐藏 size is at least 4096. At 1B-规模 the overhead hurts more than the gain helps.

当not to:

- Fine-tuning an existing pre-trained 稠密 模型. The MTP module is not 训练后的.
- Research 模型 where you want a clean 基线 to compare against. MTP changes the 架构.

## 交付成果

这lesson produces `outputs/skill-mtp-planner.md`. Given a 预训练 run specification (模型 size, 数据, 计算), it returns a plan for integrating MTP: number of depths D, `lambda` 调度, 内存 overhead, and the 推理-time speculative-decoding wiring.

## 练习

1. 运行`code/main.py`. Show the per-depth 损失 decreases monotonically as the synthetic 信号 strengthens. Modify the synthetic to use a fixed pattern and verify both depth-1 and depth-2 losses converge.

2. 计算 the 参数 overhead for a 稠密 70B 模型 (隐藏 8192, 80 层) with D=1 MTP module. Compare to the DeepSeek-V3 reported 14B overhead. Explain why DeepSeek's number is higher: the MTP transformer 块 inherits the same MoE structure, inflating the per-module 参数 count.

3. Implement D=2 in the toy: add a second MTP module that takes h^(1) and predicts `t_{i+2}`. Verify the joint 损失 and the 参数 accounting match the DeepSeek paper's equations 19-21.

4. Switch the toy to 并行 MTP (Gloeckle-style): add D 输出 头 on top of the main 隐藏 状态, each predicting a different offset. Measure how the losses per 深度 compare to the sequential version on the same synthetic 信号. The sequential version should produce lower depth-k 损失 for k > 1 because it conditions on the intermediate predictions.

5. 使用the 训练后的 MTP module as an EAGLE-style draft: call module k to propose `t_{i+k}` at 推理. Measure the acceptance 速率 of these draft 词元 against the main 模型's predictions on a held-out 序列. If you hit 50%+ on the toy, you have reproduced the empirical MTP-as-draft property.

## Key Terms

|Term|What people say|What it actually means|
|------|----------------|------------------------|
|MTP module|"Extra 损失 块"|A small transformer 块 plus projection that predicts a 词元 `k` positions ahead of the main 模型|
|预测 深度|"Which offset"|The integer `k` such that module `k` predicts `t_{i+k}` from prefix through position `i`|
|并行 MTP|"Gloeckle-style"|D independent 头 on the same backbone 隐藏 状态, no 条件式 链|
|Sequential MTP|"DeepSeek-V3 风格"|Each module conditions on the previous 深度's 隐藏 状态 plus the next 词元's 嵌入; preserves 因果 链|
|Shared 输出 头|"Reuse the main 头"|The MTP modules call the main 模型's LM 头, not a separate 输出 projection|
|Shared 嵌入|"Reuse the main table"|Same 词表 嵌入 table is used everywhere; no duplicate 参数|
|Projection matrix M_k|"Combine 隐藏 + next-token"|An `h x 2h` linear 层 that folds the previous 隐藏 状态 and the target-token 嵌入 into the next 深度's 输入|
|Joint 损失 L_MTP|"Averaged extra losses"|Arithmetic mean of per-depth cross-entropy losses, scaled by `lambda`|
|Acceptance 速率 at 深度 1|"How often MTP draft is right"|The 速率 at which the D=1 MTP module's top-1 预测 equals the main 模型's top-1 预测; 80%+ on DeepSeek-V3|
|Lambda weighting|"Extra-loss importance"|Per-depth 扩展 factor; 0.3 at start of 训练, 0.1 later on DeepSeek-V3|

## 延伸阅读

- [DeepSeek-AI — DeepSeek-V3 Technical Report (arXiv:2412.19437)](https://arxiv.org/abs/2412.19437) — the full sequential MTP 描述 (Section 2.2), including the joint-loss equations and the 1.8× speedup at 推理
- [Gloeckle et al. — Better & Faster Large Language Models via Multi-token Prediction (arXiv:2404.19737)](https://arxiv.org/abs/2404.19737) — the 并行 MTP 基线 DeepSeek's design improves on
- [DeepSeek-V3 model card on Hugging Face](https://huggingface.co/deepseek-ai/DeepSeek-V3) — 685B total (671B main + 14B MTP), deployment notes
- [Leviathan et al. — Fast Inference from Transformers via Speculative Decoding (arXiv:2211.17192)](https://arxiv.org/abs/2211.17192) — the speculative-decoding framework MTP fits into
- [Li et al. — EAGLE-3 (arXiv:2503.01840)](https://arxiv.org/abs/2503.01840) — EAGLE's 2025 draft 架构, the counterpart MTP competes with
