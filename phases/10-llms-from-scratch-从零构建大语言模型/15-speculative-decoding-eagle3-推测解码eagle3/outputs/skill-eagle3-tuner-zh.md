---
name: eagle3-tuner-zh
description: 选择and tune a 推测解码 strategy (vanilla / Medusa / EAGLE-1/2/3 / lookahead) for a new 推理 workload.
version: 1.0.0
phase: 10
lesson: 15
tags: [speculative-decoding, eagle, eagle-3, medusa, inference, vllm, sglang, tensorrt-llm]
---

给定a 生产 推理 目标 (verifier 模型, 批次 size, 序列 length profile, 目标 p50/p99 decode 延迟, accelerator, expected alpha range from telemetry, 任务 mix), recommend a speculative-decoding strategy and tuning 参数. The recommendation must preserve the verifier's 输出 分布 exactly — no 质量 tradeoff is acceptable without explicit sign-off.

Produce:

1. Draft family. Pick from vanilla, Medusa, EAGLE-1, EAGLE-2, EAGLE-3, or lookahead. Justify using alpha telemetry (or a calibrated estimate), 训练 成本 available (none, small SFT, full 60B+ 词元 run), and whether the verifier ships with a published draft (EAGLE-3 checkpoints exist for Llama 3.1/3.3, DeepSeek-V3, Qwen 2.5, Qwen 3).
2. Draft length N. Pick the integer N that minimizes expected wall time per 词元 given alpha and draft-to-verifier 成本 比例 c: minimize (1 + N*c) / ((1 - alpha^(N+1)) / (1 - alpha)). Show the work for three candidate N values around the optimum.
3. Tree search 参数 if EAGLE-2/3. Pick tree 深度 and branching factor to stay within 内存 预算. Default to 深度 3, branching (4, 2, 2) for 批次 <=8, 深度 2 (4, 2) for 批次 16-64, and no tree for 批次 >64.
4. Temperature gating. When temperature > 0.8, alpha collapses. Recommend disabling spec decode above a calibrated 阈值, or switching to a wider tree with lower per-node branching.
5. KV rollback plan. Name the specific KV 缓存 implementation (vLLM's scratch buffer vs TensorRT-LLM's logical-length per-sequence) and confirm it supports batched rejection at the 目标 concurrency.

Hard rejects:
- Any recommendation that changes the verifier's 输出 分布 (e.g., approximate spec-decode, relaxed rejection).
- Spec decode at 批次 1 on a single small 模型 where draft 成本 exceeds verifier 成本 saved.
- EAGLE with a draft checkpoint 训练后的 against a different 分词器 or base 模型 revision than the verifier.
- Running spec decode without KV rollback — will silently corrupt subsequent 词元.

Refusal rules:
- 如果alpha telemetry is unavailable AND the 任务 mix is high-temperature creative writing, refuse the recommendation and request a calibration run first.
- 如果the verifier is smaller than 7B 稠密 参数, recommend disabling spec decode rather than picking a strategy.
- 如果the serving stack does not support the chosen draft family (e.g., vLLM version without EAGLE-3), downgrade to EAGLE-2 rather than asking the 用户 to rebuild the stack.

输出： a one-page recommendation listing draft family, N, tree shape (if applicable), KV rollback confirmation, and expected speedup range. End with an "alpha telemetry plan" paragraph naming the exact logging hooks the 用户 must add to their 推理 服务器 to verify the recommendation in the first week of 生产.
