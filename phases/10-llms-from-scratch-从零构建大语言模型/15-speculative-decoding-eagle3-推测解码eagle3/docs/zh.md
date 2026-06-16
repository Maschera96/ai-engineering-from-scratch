# 推测解码 and EAGLE-3

> Phase 7 · Lesson 16 proved the math: the Leviathan rejection rule preserves the verifier's 分布 exactly. This lesson is the training-stack view of 2026 生产 推测解码. EAGLE-3 turned the draft 模型 from a cheap approximation into a purpose-built tiny network 训练后的 on the verifier's own 隐藏 states, then added a training-time test 循环 that aligns its 训练 and 推理 distributions. Result: 3× to 6.5× end-to-end speedup, accepted per-token rates above 0.9 on chat, no distributional tradeoff. Every 生产 推理 stack in 2026 ships it by default.

**类型：** Build
**语言：** Python (stdlib)
**先修：** Phase 7 · 16 (推测解码 math), Phase 10 · 12 (推理 优化)
**时间：** 约 75 分钟

## 学习目标

- 状态 the Leviathan theorem in one sentence and prove that the speculative 循环 produces 样本 identically 分布式 to the verifier.
- Walk the two-year progression from vanilla spec-decoding (Leviathan 2023) through EAGLE, EAGLE-2, and EAGLE-3 and name the exact limitation each 步骤 removed.
- 计算 expected speedup from acceptance 速率 `α` and draft-to-verifier 成本 比例 `c`, and choose the optimal draft length `N` for each regime.
- Implement the full speculative 循环 from scratch: draft, verify, reject-sample from the residual, roll the KV 缓存 back on rejection, emit the bonus 词元 on full acceptance.

## 问题

Autoregressive decoding on a 70B 模型 runs at maybe 35 词元 per second on an H100. The GPU is nowhere near saturated. 内存 bandwidth is the ceiling: every 词元 loads 70B of 权重 from HBM, does one 步骤 of arithmetic, and produces one float. The 计算 units sit mostly idle.

Speculative decoding turns that into a throughput problem you can actually solve. A cheap draft proposes `N` 词元 in `N` small forward passes. The verifier runs once on the prefix plus all `N` drafts. If the verifier's 分布 at position `i` agrees with the draft (in a statistical sense we will make precise), we accept; if not, we reject and 样本 a correction from the residual 分布. A single big-model forward produces up to `N+1` accepted 词元 instead of one.

这个theorem that matters is Leviathan, Kalman, Matias (ICML 2023): the 输出 分布 is identical to what 采样 from the verifier directly would have produced. Not approximately. Identically. This is the entire 原因 推测解码 is acceptable in 生产 — it is a pure 延迟 优化 with no 质量 tradeoff.

What Phase 7 · Lesson 16 gave you was the math. What this lesson gives you is the 训练 stack. A good draft is worth 2× more speedup than a cheap draft. EAGLE, EAGLE-2, and EAGLE-3 (Li et al., 2024–2025) turned "draft = smaller version of the same 模型" into a precise engineering discipline. 2026 生产 推理 servers default to EAGLE-3.

## 概念

### The invariant: Leviathan rejection 采样

Let `p(t)` be the draft's 分布 for the next 词元 given some prefix, and `q(t)` be the verifier's. 样本 a draft 词元 `d ~ p`. Accept with 概率 `min(1, q(d) / p(d))`. On reject, 样本 from the residual 分布 `(q - p)_+ / ||(q - p)_+||_1`. The resulting 样本 are 分布式 according to `q`. This is true regardless of how bad `p` is — the worse it is, the more often you reject, but the 输出 remains exact.

Stack `N` of these calls back to back using one verifier forward pass on `prefix + d_1 + ... + d_N`. The verifier returns `q_1, q_2, ..., q_{N+1}` simultaneously. Walk left to right. On the first rejection at position `j`, 样本 from `residual(q_j, p_j)` and stop. On full acceptance, 样本 one bonus 词元 from `q_{N+1}`.

### What determines speedup

Let `α` be the expected acceptance 速率 per drafted 词元. Let `c = cost(draft) / cost(verifier)` be the 成本 比例. The expected number of accepted 词元 per verifier forward is:

```text
E[accepted] = (1 - α^(N+1)) / (1 - α)
```

这个expected total wall time per accepted 词元 is `(N * c + 1) / E[accepted]`. Minimize that with respect to `N` and you get the sweet spot. For `α = 0.8, c = 0.05`: optimal `N` is around 5–7, speedup is 3.2×. For `α = 0.95, c = 0.02`: optimal `N` is around 8–10, speedup pushes 5×.

这个single biggest lever is `α`. Going from `α = 0.6` (vanilla draft) to `α = 0.9` (EAGLE-3) at fixed `N = 5` takes you from 2.2 expected accepted 词元 per verifier forward to 4.1. Nearly 2× more throughput from the same verifier.

### The two-year progression

**Vanilla speculative (Leviathan, 2023).** Draft 模型 is an independently 训练后的 smaller LLM from the same family. Easy to wire up, `α ≈ 0.6`, speedup around 2× at best.

**EAGLE-1 (Li et al., 2024).** Draft is a tiny transformer — typically one or two 层 — that takes the verifier's last-layer 隐藏 状态 as 输入 and predicts the next 词元 directly. Because the draft sees the verifier's 特征 representation, its 分布 is much closer to the verifier's. `α` climbs to 0.7–0.8.

**EAGLE-2 (Li et al., 2024).** Adds a dynamic draft tree: instead of proposing a single 序列 of `N` 词元, propose a small tree of candidates, 分数 each with the verifier in one forward pass (tree 注意力), and walk the highest-probability path. Draft length becomes adaptive per 步骤. `α` per accepted-path 词元 climbs above 0.85.

**EAGLE-3 (Li et al., 2025, NeurIPS).** Two more changes. First, drop the 特征-预测 损失 entirely — EAGLE-1/2 训练后的 the draft to match the verifier's 隐藏 states, which caps how much 数据 helps. EAGLE-3 trains directly on 词元 预测. Second, training-time test (TTT): during draft 训练, feed the draft's own previous predictions back as inputs over multiple 步骤, the same way it operates at 推理. This aligns the 训练 and test distributions and stops 错误 accumulation. Measured speedup: up to 6.5× on chat, 38% throughput improvement at 批次 64 in SGLang on H100.

### KV 缓存 rollback

Verification extends the verifier's KV 缓存 by `N` entries in one pass. If rejection happens at position `j`, the 缓存 contents past position `j-1` are now wrong. Two common implementations: write to a scratch buffer and commit on acceptance (vLLM, TensorRT-LLM), or keep a physical KV 缓存 plus a logical length and truncate on reject. Either way, the rollback 成本 is bytes per 层 per 头, which is negligible next to the forward-pass 成本.

For EAGLE-2 tree search, the verifier runs 注意力 with a non-causal 掩码 that respects tree 拓扑. The engineering is fiddly but the computation is a standard flash-attention call with a custom 掩码.

### Draft architectures in 2026

|Strategy|Draft type|`α`|Speedup|训练 成本|
|----------|-----------|-----|---------|---------------|
|Vanilla|Separate small LLM|0.55-0.70|1.8-2.3×|None (reuse existing small 模型)|
|Medusa|Extra LM 头 on verifier|0.65-0.75|2-3×|~1B SFT 词元|
|EAGLE-1|1-层 transformer on 隐藏 states|0.70-0.80|2.5-3×|~60B 词元|
|EAGLE-2|EAGLE-1 + dynamic draft tree|0.80-0.88|3-4×|~60B 词元|
|EAGLE-3|Multi-layer 特征 fusion + TTT|0.88-0.92|3.5-6.5×|~60-200B 词元|
|Lookahead|No draft (Jacobi iteration)|N/A|1.3-1.6×|None|

In 2026 生产: vLLM and SGLang default to EAGLE-3 when available, EAGLE-2 otherwise. TensorRT-LLM has the fastest Medusa path for Meta and NVIDIA public 模型. llama.cpp ships vanilla draft for CPU deployments.

## 动手构建

See `code/main.py`. This is the full Leviathan speculative 循环 with all the pieces: draft-of-N, verifier 并行 pass, per-position rejection, residual 采样, bonus 词元, KV rollback, and empirical verification that the 输出 分布 matches direct 采样 from `q`.

### 步骤 1: the rejection rule

```python
def accept(q_prob, p_prob, u):
    if p_prob <= 0:
        return True
    return u < min(1.0, q_prob / p_prob)
```

### 步骤 2: residual 分布

```python
def residual(q, p):
    raw = [max(0.0, qi - pi) for qi, pi in zip(q, p)]
    s = sum(raw)
    if s == 0:
        return list(q)
    return [r / s for r in raw]
```

### 步骤 3: a full speculative 步骤

这个`spec_step` 函数 drafts `N` 词元 from `p`, then verifies all of them in one 并行 `q` 评估. For each drafted 词元 it applies the rejection rule, and on the first rejection it 样本 the correction from the residual. If everything accepts, it emits a bonus 词元 from `q_{N+1}`.

### 步骤 4: KV rollback bookkeeping

这个simulator tracks a logical `kv_length` per worker. On acceptance of `k` drafts, `kv_length += k`. On a rejection at position `j`, the 缓存 is already written past `j`, but the logical length is set to `prefix_length + j + 1` — one past the correction 词元. Subsequent reads truncate to the logical length.

### 步骤 5: the Leviathan check

运行50,000 speculative 步骤. Count the empirical 分布 of accepted 词元. Compare to 50,000 direct 样本 from `q`. The chi-square statistic should be well under the critical value. The theorem passes in practice.

### 步骤 6: speedup vs. α

Sweep the draft 质量 by perturbing `p` away from `q` at different amplitudes. Measure `α`, then plot expected 词元 per verifier call as a 函数 of `α` and `N`. The code prints a table showing how EAGLE-3-class draft 质量 (`α ≈ 0.9`) unlocks 4–5 词元 per verifier call.

## 实际使用

Production-level `vllm serve` with EAGLE-3:

```bash
vllm serve meta-llama/Llama-3.3-70B-Instruct \
  --speculative-config '{
    "model": "yuhuili/EAGLE3-LLaMA3.3-Instruct-70B",
    "num_speculative_tokens": 5,
    "method": "eagle3"
  }'
```

SGLang with EAGLE-3 at 批次 64 on H100: roughly 1.38× more throughput than batch-64 vanilla decoding, per the EAGLE-3 paper.

当to reach for 推测解码:

- Any interactive chat workload where p50 延迟 matters more than peak throughput.
- Code 生成 and 结构化 输出 (JSON, SQL). `α` is above 0.9 because the 目标 分布 is highly predictable.
- Long-form 生成 (thousands of 词元). The amortized speedup keeps paying.

当not to:

- Very small 模型 (< 3B). The draft is not that much cheaper than the verifier.
- Tiny batch-1 CPU deployments. 内存 overhead of the draft 模型 may not be worth it.
- Very-high-temperature creative 采样 where `α` collapses.

## 交付成果

这lesson produces `outputs/skill-eagle3-tuner.md`. Given an 推理 workload (模型, 批次 size, 目标 延迟, 任务 profile), it recommends a speculative-decoding strategy and tuning 参数 (draft family, `N`, tree 深度, temperature-aware switching).

## 练习

1. 运行`code/main.py`. Confirm the chi-square statistic on the Leviathan 分布 check stays below the 95% critical value on 50,000 样本.

2. Sweep `N` from 1 to 10 with `α` held at 0.9 and `c` held at 0.04. Plot expected 词元 per verifier call and actual wall time per 词元. Find the `N` that minimizes wall time. Explain the shape of the 曲线.

3. Modify the code to simulate EAGLE-2 tree search: at each 步骤, the draft proposes a tree of shape `[2, 2, 2]` (eight candidate paths). The verifier runs once, and the highest-probability accepted path wins. 计算 `α` per leaf and total 词元 per verifier call. Compare to linear-chain spec-decoding at equivalent 计算.

4. Implement a batched KV rollback simulator for two concurrent sequences. 序列 A has all drafts accepted; 序列 B rejects at position 2. Show that the correct `kv_length` is updated per 序列 and that no work is wasted.

5. Read the EAGLE-3 paper's Section 4 (Training-时间 Test). Explain in two sentences why naive draft 训练 without TTT suffers from exposure 偏差, and why feeding the draft its own predictions during 训练 fixes it. Connect this to the scheduled-sampling literature in seq2seq.

## Key Terms

|Term|What people say|What it actually means|
|------|----------------|------------------------|
|Leviathan rule|"min(1, q over p)"|Bernoulli accept/reject with 概率 `min(1, q(d)/p(d))`, preserves the verifier 分布 exactly when you 样本 from the residual on rejection|
|Residual 分布|"(q minus p) plus, normalized"|`(q - p)_+` clamped at zero and renormalized — the correct 分布 to 样本 from on rejection|
|Acceptance 速率 α|"how often the draft is right"|Expected per-token Bernoulli-success 概率 under the rejection rule; governs all speedup math|
|EAGLE-1|"hidden-state draft"|Tiny transformer draft conditioned on the verifier's last-layer 隐藏 状态 (Li et al., 2024)|
|EAGLE-2|"dynamic draft tree"|EAGLE-1 plus a tree of candidate continuations scored with tree 注意力 in one verifier pass|
|EAGLE-3|"training-time test"|Drops the 特征-预测 损失, trains on direct 词元 预测 with the draft fed its own outputs during 训练|
|Training-time test (TTT)|"exposure 偏差 fix"|Run the draft autoregressively during 训练 so 训练 and test 输入 distributions match — the direct analog of scheduled 采样|
|KV rollback|"undo rejected drafts"|Bookkeeping that resets the verifier's KV 缓存 to the accepted-prefix length after a rejection|
|Bonus 词元|"the free one"|When all `N` drafts accept, 样本 one extra from `q_{N+1}` at no additional verifier 成本|
|Tree 注意力|"verify many candidates at once"|注意力 with a non-causal 掩码 that respects the 拓扑 of a draft tree; computes `q_i` for every 节点 in the tree in one forward pass|

## 延伸阅读

- [Leviathan, Kalman, Matias — Fast Inference from Transformers via Speculative Decoding (arXiv:2211.17192, ICML 2023)](https://arxiv.org/abs/2211.17192) — the foundational paper and equivalence theorem
- [Chen et al. — Accelerating Large Language Model Decoding with Speculative Sampling (arXiv:2302.01318)](https://arxiv.org/abs/2302.01318) — concurrent independent introduction with a clean proof
- [Li et al. — EAGLE: Speculative Sampling Requires Rethinking Feature Uncertainty (arXiv:2401.15077)](https://arxiv.org/abs/2401.15077) — EAGLE-1, hidden-state-conditioned draft
- [Li et al. — EAGLE-2: Faster Inference of Language Models with Dynamic Draft Trees (arXiv:2406.16858)](https://arxiv.org/abs/2406.16858) — dynamic tree search
- [Li et al. — EAGLE-3: Scaling up Inference Acceleration via Training-Time Test (arXiv:2503.01840, NeurIPS 2025)](https://arxiv.org/abs/2503.01840) — the 2026 生产 default
- [Cai et al. — Medusa: Multiple Decoding Heads (arXiv:2401.10774)](https://arxiv.org/abs/2401.10774) — 替代方案 draft-free approach
- [vLLM Speculative Decoding documentation](https://docs.vllm.ai/en/latest/features/spec_decode.html) — canonical 生产 参考 with all strategies wired up
