---
name: speculative-tuning-zh
description: Profile a decode workload and pick draft 模型, draft length K, temperature gate, and 备选方案 策略 for 推测解码.
version: 1.0.0
phase: 10
lesson: 25
tags: [speculative-decoding, draft-model, alpha, throughput, inference, decode-latency]
---

给定the 目标 模型 (size, family, 分词器), the workload telemetry (任务 mix, prompt-vs-decode 词元 比例, p50/p99 decode 延迟, accelerator and HBM headroom, average 批次 size, 采样 temperature 分布), and the available draft checkpoints, 输出：

1. Draft choice. Pick from same-family small (Llama-3.2-1B for Llama-70B), distilled draft (Qwen3-0.6B-spec), Medusa 头 bolted on the 目标, or "no spec decode" if no draft is closer than 30 percent FLOP 成本 比例. Confirm 分词器 match against the 目标 byte-for-byte; refuse a mismatched 分词器.
2. Draft length K. Argmax of E[词元] / (1 + K x c) where c is the draft-to-target 成本 比例. Show the work for K in 2, 3, 4, 5, 6 using the measured alpha from a calibration run on 5_000 词元 of in-distribution 数据. Default K=4 for chat, K=6 for code, K=2 for high-temperature creative writing.
3. Temperature gate. Set a temperature 阈值 above which spec decode is disabled. Default 0.8; lower to 0.6 if the calibration shows alpha collapsing earlier. Reject any temperature gate that depends on per-request inspection that adds more than 50 microseconds.
4. Tree 预算. If the serving stack supports tree drafting, pick a small fixed tree (深度 2, branch 3-2) for 批次 under 8; flat 链 for 批次 over 32. 状态 the verifier's KV scratch size in bytes and confirm it fits in HBM headroom.
5. 备选方案 策略. Name the 指标 (sliding-window measured alpha over the last 1_000 verifies) and the 阈值 (alpha under 0.4) at which the 服务器 drops back to plain autoregressive decode for that request stream. Include the per-request lifetime of the 备选方案 decision.

拒绝spec decode at 批次 size above the point where the verifier is compute-bound. Above that point the unused FLOPs the speculator is meant to soak up no longer exist; throughput drops. Refuse spec decode for any 任务 family with measured alpha under 0.4; the draft overhead dominates and wall-clock 延迟 gets worse. Refuse a draft that has not been validated on a held-out 1_000-词元 样本 against the 目标: an unvalidated draft is a silent KL drift.

Example 输入: "Llama-3.3-70B on 8xH100, chat workload, 批次 16, p50 decode 28 ms, p99 60 ms, temperature 分布 mean 0.4 / max 1.2, calibration shows alpha 0.78 on chat, 0.61 on code."

Example 输出：
- Draft: Llama-3.2-1B-Instruct-spec. Same 分词器, same family, 比例 c approx 0.03.
- K: 4. E[词元/verify] = 3.4 chat, 2.5 code. K=5 gains 0.1 词元 chat and pays 0.03 extra c; reject.
- Temperature gate: 0.8. Above 0.8 alpha drops below 0.45 on the calibration set.
- Tree 预算: 深度 2 branch (3, 2). KV scratch 480 MB at 批次 16 fits.
- 备选方案: sliding-window alpha over last 1_000 verifies under 0.40 disables spec decode for that stream for 30 s, then probes again.
