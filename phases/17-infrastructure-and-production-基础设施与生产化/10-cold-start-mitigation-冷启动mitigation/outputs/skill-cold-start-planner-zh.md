---
name: cold-start-planner-zh
description: 选择and stack cold-start mitigations for Serverless LLM deployments. Budget phases (node, image, weights, engine, first forward) and match mitigations to SLA.
version: 1.0.0
phase: 17
lesson: 10
tags: [cold-start, serverless, bottlerocket, model-streamer, gpu-snapshot, warm-pool, serverlessllm]
---

给定模型 size, SLA (TTFT P99), 流量 shape (steady 对比 bursty), and budget posture, produce a cold-start mitigation 计划.

产出：

1. Cold-start budget. Break down the raw cold-start path (node provision, image pull, weights to HBM, engine init, first forward). Use 2026 nominal seconds for the stated 模型 size.
2. Layer selection. Pick the minimum number of layers that brings total below the SLA: pre-seeded image (L1), 模型 streamer (L2), GPU snapshot (L3), 预热池 (L4), tiered loading (L5). 说明理由 each layer against 这个具体 phase it attacks.
3. Warm-pool sizing. State `min_workers` for the primary path. If SLA is TTFT P99 < 60s on a 70B+ 模型, make 预热池 mandatory regardless of 成本.
4. 成本 estimate. Monthly GPU 成本 for the chosen warm-pool and the expected number of cold starts per day.
5. 尾部 策略. What happens to the first user on a fresh replica ： do they get queued to a warm replica, or do they pay the cold-start tax? Name 一个具体策略 (e.g., "路由 first 请求 to any warm replica within 10s; fall through to cold").
6. Failure mode. What happens if a warm replica dies mid-session. Is recovery automatic (live 迁移), or is it 一个冷启动 on the next 请求?

硬性拒绝：
- Proposing "just add 预热池" without computing 这个每月成本.
- Claiming a mitigation without 一个具体 phase it attacks (e.g., "use Bottlerocket" without saying it eliminates the 180s image pull).
- Ignoring the per-GPU-topology constraint on GPU snapshots ： if 这个平台 migrates SKU, snapshots are invalid.

拒绝规则：
- If SLA is TTFT P99 < 5s on a fresh 70B 冷启动 with no 预热池, 拒绝 ： mathematically impossible at 2026 基础设施 speeds.
- If budget forbids 预热池 but SLA 需要 sub-30s 冷启动, name 这个平台-具体 fix (Modal GPU snapshots, Baseten pre-warming) 和 拒绝 to 承诺 the SLA on 一个不同平台 without it.
- If the operator asks for scale-to-zero with bursty 流量 and a 70B 模型, 拒绝 to 承诺 SLA ： the math does not work without snapshots or warm pools.

输出： a one-page 计划 listing phases, layers, `min_workers`, 每月成本, 尾部 策略, failure mode. End with the single 指标 to alert on: P99 cold-start duration over the last rolling 小时.
