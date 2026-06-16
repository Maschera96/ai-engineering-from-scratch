# 生产化中的 EAGLE-3 推测解码

> Speculative decoding pairs a fast draft 模型 with the target 模型. The draft proposes K 词元; the target verifies in a single forward; accepted 词元 are free. In 2026, EAGLE-3 is 这个生产化-grade variant ： it trains a draft head on the target 模型's hidden states rather than on raw 词元, pushing acceptance rate alpha into the 0.6-0.8 band on general chat. The right question is not "how fast is the draft" but "what is alpha on my 流量?" If alpha drops below ~0.55, speculative decoding is net negative at high concurrency because every rejected draft costs a second target forward pass. This lesson teaches you to measure alpha first and flip the flag second.

**Type:** Learn
**Languages:** Python（标准库， 玩具 acceptance-rate模拟器)
**Prerequisites:** Phase 17 · 04 (vLLM Serving Internals), Phase 10 · 18 (Multi-Token Prediction)
**Time:** ~60 分钟

## 学习目标

- Name the three generations of speculative decoding and 解释 what EAGLE-3 changes from EAGLE-2 and from a classic draft 模型.
- Define acceptance rate alpha, compute expected speedup from alpha and K (draft length), and identify 这个盈亏平衡 alpha for your target concurrency.
- 解释 why speculative decoding is opt-in (not 默认) in vLLM 2026 and why turning it on without measuring alpha is 一个生产化 anti-模式.
- Write a measurement 计划: which 基准, which 提示词 distribution, which concurrency point, which 指标 to gate on.

## 问题

Decode is 内存-bound. On an H100 running Llama 3.3 70B FP8, each decoded 词元 reads ~140 GB/s of weights and emits one 词元. The GPU compute is almost 空闲 during decode ： the bottleneck is HBM 带宽, not matmul 吞吐量.

Speculative decoding exploits the gap. Generate K candidate 词元 with a cheap draft 模型, then ask the target 模型 to verify all K in a single forward pass. Each verified 词元 is effectively free (amortized into 一个批处理-of-K forward the target would have had to do anyway).

The classic draft-模型 approach uses a smaller 模型 of the same family (Llama 3.2 1B drafting for Llama 3.3 70B). It works but acceptance rate is mediocre ： the smaller 模型 distribution diverges from the target. EAGLE, then EAGLE-2, then EAGLE-3 train a light draft head 直接 on the target 模型's internal states, so the draft's distribution tracks the target much more closely. That is why alpha goes from 0.4 with draft-模型 to 0.6-0.8 with EAGLE-3.

The catch: EAGLE-3 is opt-in in vLLM 2026. `speculative_config` must be set explicitly. No flag, no acceleration. 团队 that flip it on without measuring alpha on their real 流量 often see 尾部 延迟 get worse, not better.

## 概念

### 推测解码真正带来什么

Without spec decode, per-词元 成本 is one target forward. With spec decode at draft length K and acceptance alpha, expected 词元 per target forward is `1 + K * alpha`. The speedup is `(1 + K * alpha) / (1 + epsilon)` where epsilon is draft-plus-verify overhead. For K=5, alpha=0.7: `(1 + 5*0.7) / (1 + 0.1) = 4.5 / 1.1 = 4.1x`. Real-world numbers cluster around 2-3x because alpha is rarely that high on 生产化 流量 and epsilon grows at high 批处理 size.

### 为什么 alpha 是唯一关键指标

Rejected 词元 do not disappear ： they force a second target forward for the first rejected 词元. On 一个工作负载 where alpha drops to 0.4, you pay draft overhead plus verification plus re-roll. At high concurrency (say 256 concurrent), the decode 批处理 is already large enough that 这个内存-带宽 gap between "target alone" 和 "target with verify" shrinks. Below alpha 0.55 on most 2026 hardware, spec decode is net negative.

Alpha 变化 by 工作负载. On ShareGPT-style general chat, EAGLE-3 trained on ShareGPT hits 0.6-0.8. On domain-具体 流量 (code, medical, legal) the draft head trained on general data drops to 0.4-0.6. Training a domain-具体 draft head recovers alpha ： it is a light, quick training job compared to target finetuning.

### EAGLE 代际速览

- **Classic draft 模型**: small 模型 of same family. Alpha 0.3-0.5. 基础设施 简单 ： two 模型 loaded, draft runs K forwards per target forward.
- **EAGLE-1 (2024)**: single draft head trained on target hidden states (last layer). Alpha ~0.5-0.6. Small param overhead on top of target.
- **EAGLE-2 (2025)**: adaptive draft length and tree-based drafts (verify multiple branches in one target pass). Alpha ~0.6-0.7. More complex draft 调度器.
- **EAGLE-3 (2025-2026)**: draft head trained on multiple target layers (not just last), better alignment. Alpha ~0.6-0.8 on general chat.

### 2026 年生产化配方

1. Ship target 模型 plain. Measure baseline TTFT, ITL, 吞吐量 at target concurrency.
2. Enable EAGLE-3 draft via vLLM `speculative_config`. Re-run 这个基准.
3. Log acceptance rate alpha. vLLM V1 reports this as `spec_decode_metrics.accepted_tokens_per_request`. Divide by requested draft length to get alpha.
4. If alpha < 0.55 on 生产化 流量 distribution, disable spec decode or train a domain-具体 EAGLE-3 draft.
5. At 生产化 concurrency, re-run. Confirm P99 ITL did not get worse.

### 生产化陷阱：P99 尾部

均值 ITL drops with spec decode. P99 can get worse if you do not tune. Rejected drafts trigger a two-pass sequence (draft + verify-fail + reroll). Under full 批处理, those two passes serialize. Watch P99 ITL, not P50.

### EAGLE-3 已经部署在哪里

Google deployed speculative decoding in AI Overviews in 2025 (same 质量, faster response). vLLM V1 ships `speculative_config` as the documented interface; N-gram GPU speculative decoding in V1 is the variant compatible with chunked prefill. SGLang supports EAGLE-3 as the recommended draft path for prefix-heavy 工作负载.

### 一行算清盈亏平衡

Expected speedup: `S(alpha, K) = (1 + K*alpha) / (1 + verify_overhead)`. Setting `S = 1` solves for alpha: `alpha_breakeven = verify_overhead / K`. For typical verify_overhead ~0.15 and K=5: `alpha_breakeven = 0.03`. But that is the raw decode math. At high concurrency the verify overhead rises and the decode 批处理 already amortizes 内存 reads across sequences, so effective alpha_breakeven climbs to ~0.45-0.55 in practice.

### 什么时候不要使用推测解码

- 批处理-1 offline generation where 延迟 does not matter. Use plain target.
- Very short 输出 (under 50 词元). Draft overhead and verify 成本 dominate.
- Specialized domains without a domain-trained draft head. Alpha too low.
- vLLM v0.18.0 plus draft-模型 spec decode plus `--enable-chunked-prefill`. This combination does not compile. The documented exception is N-gram GPU spec decode in V1.

## 使用它

`code/main.py` simulates a decode loop with and without speculative decoding across a range of alpha values and draft lengths K. It prints 这个盈亏平衡 alpha, measured speedup, 和 尾部 behavior. Run it on several (alpha, K) combinations to see exactly where speculative decoding stops paying.

## 交付它

This lesson 产出 `outputs/skill-eagle3-rollout.md`. Given a target 模型, 流量 distribution description, and concurrency target, it 产出 a staged EAGLE-3 rollout 计划 ： 基准 baseline, enable config, measure alpha, gate on alpha >= 0.55, watch P99 ITL.

## 练习

1. Run `code/main.py`. At K=5, what alpha do you need for a 2x speedup? For a 3x speedup? How sensitive is that to verify_overhead?
2. Imagine 生产化 流量 splits 70% general chat, 30% code. General chat hits alpha 0.7 with EAGLE-3 trained on ShareGPT; code hits alpha 0.4. What is blended alpha and is spec decode net-positive?
3. Read the vLLM `speculative_config` documentation. Name the three modes (draft 模型, EAGLE, N-gram) and which one is compatible with chunked prefill.
4. You see 均值 ITL drop 25% after enabling EAGLE-3 but P99 ITL went up 15%. Diagnose and propose a mitigation.
5. Compute 这个内存 成本 of the EAGLE-3 draft head for Llama 3.3 70B. How does it 比较 to running Llama 3.2 1B as a classic draft?

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| Speculative decoding | "draft plus verify" | Propose K 词元 with a cheap 模型, verify all K in one target forward |
| Acceptance rate alpha | "spec accept rate" | Fraction of draft 词元 accepted by the target; the only 指标 that matters |
| Draft length K | "spec k" | How many 词元 the draft proposes per target forward; typical 4-8 |
| Verify overhead epsilon | "spec overhead" | Extra 成本 to verify-and-reroll 对比 a plain target forward; grows with 批处理 |
| EAGLE-3 | "latest EAGLE" | 2025-2026 variant; trains draft head on multiple target layers; alpha 0.6-0.8 on general chat |
| `speculative_config` | "vLLM spec config" | The explicit opt-in in vLLM V1; no 默认 means no acceleration |
| N-gram spec decode | "N-gram draft" | GPU-side draft using N-gram lookups in 这个提示词; chunked-prefill-compatible |
| 盈亏平衡 alpha | "no-op alpha" | Alpha at which spec decode gives zero speedup; watch this at 生产化 concurrency |
| Rejected-draft two-pass | "reroll 成本" | Two target forwards when drafts reject; drives P99 尾部 |

## 延伸阅读

- [vLLM — Speculative Decoding docs](https://docs.vllm.ai/en/latest/features/spec_decode/) ： 权威 source on `speculative_config` and chunked-prefill compatibility in V1.
- [vLLM Speculative Config API](https://docs.vllm.ai/en/latest/api/vllm/config/speculative/) ： the exact field set.
- [EAGLE paper (arXiv:2401.15077)](https://arxiv.org/abs/2401.15077) ： original EAGLE draft-head formulation.
- [EAGLE-2 paper (arXiv:2406.16858)](https://arxiv.org/abs/2406.16858) ： adaptive drafts and trees.
- [UC Berkeley EECS-2025-224](https://www2.eecs.berkeley.edu/Pubs/TechRpts/2025/EECS-2025-224.html) ： efficient LLM system with speculative decoding.
- [BentoML — Speculative Decoding](https://bentoml.com/llm/inference-optimization/speculative-decoding) ： 生产化 rollout checklist.
