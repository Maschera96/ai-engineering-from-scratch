# Serverless 大模型的冷启动缓解

> A 20 GB 模型 image takes 5-10 minutes (7B) to 20+ minutes (70B) to go from cold to serving. In a true Serverless world, that is not a warm-up ： it is an outage. Mitigations operate at five layers: pre-seeded node images (Bottlerocket on AWS, dual-volume arch), 模型 streaming (NVIDIA Run:ai 模型 Streamer, native in vLLM), GPU 内存 snapshots (Modal checkpoints, up to 10x faster restart), warm pools (`min_workers=1`), tiered loading (ServerlessLLM's NVMe→DRAM→HBM pipeline, 10-200x 延迟 reduction), and live 迁移 that moves 输入 词元 (KB) rather than KV 缓存 (GB). Modal publishes 2-4s cold starts as a floor; Baseten 5-10s 默认, sub-second with pre-warming. This lesson teaches you to measure, budget, and stack the five layers.

**Type:** Learn
**Languages:** Python（标准库， 玩具 cold-start path模拟器)
**Prerequisites:** Phase 17 · 02 (Inference Platform Economics), Phase 17 · 03 (GPU Autoscaling)
**Time:** ~60 分钟

## 学习目标

- Enumerate the five layers of cold-start mitigation and name one tool or 模式 at each layer.
- Compute total cold-start time as a sum of (node provision) + (weights download) + (weights load into HBM) + (engine init) for a 70B 模型.
- 解释 why live 迁移 transfers 输入 词元 (KB) not KV 缓存 (GB) and what the penalty is (recomputation).
- Name the warm-pool trade-off (pay for 空闲 GPU or accept cold-start 尾部) and the SLA threshold at which `min_workers > 0` becomes mandatory.

## 问题

Your Serverless LLM endpoint scales to zero overnight. At 8 a.m. 流量 spikes. The first 请求 waits while:

1. Karpenter provisions a GPU node: 45-60s.
2. The container pulls a 30 GB image with weights: 120-300s.
3. The engine loads weights into HBM: 45-120s depending on 模型 size and storage speed.
4. vLLM or TRT-LLM initializes CUDA graphs, KV 缓存 pool, tokenizer: 10-30s.

Total: 220-510s (roughly 3-8 minutes) before one 词元 comes back. Your SLA is 2s. You ship a warm-pool (`min_workers=1`) and the problem seems to vanish ： but now you pay for one 空闲 GPU 24x7. If your service has 5 products each with one warm replica, that's 5 × 24 × 30 = 3,600 GPU-hours/月 whether or not a single user called.

Cold-start mitigation is how to keep the Serverless economics while approximating 这个延迟 of always-on.

## 概念

### Layer 1 ： pre-seeded node images (Bottlerocket)

On AWS, Bottlerocket's dual-volume architecture separates OS from data. Snapshot the data volume with your container image pre-pulled; reference the snapshot ID in your `EC2NodeClass`. New nodes boot with weights already on local NVMe ： steps 2 and part of 3 vanish. Works with Karpenter natively. Typical 节省: 2-4 minutes per 冷启动 for large 模型.

Equivalent on GCP: custom VM images with pre-baked container layers. On Azure: managed disk snapshots with the same 模式.

### Layer 2 ： 模型 streaming (Run:ai 模型 Streamer)

Instead of loading the full file before answering the first 请求, stream weights into GPU 内存 layer-by-layer and start processing as soon as the first transformer block is resident. The NVIDIA Run:ai 模型 Streamer ships native in vLLM 2026. Works with S3, GCS, and local NVMe. Cuts weight-load time roughly in half for large 模型 by overlapping I/O with compute setup.

### Layer 3 ： GPU 内存 snapshots (Modal)

Modal takes a checkpoint of the GPU state (weights, CUDA graphs, KV 缓存 区域) after first load. Subsequent restarts deserialize 直接 into HBM ： 10x faster than re-initializing. This is the closest thing to "boot a warm GPU in 2 seconds." Trade-off: snapshots are per-GPU-topology, so if Karpenter migrates you to a different SKU, you re-checkpoint.

### Layer 4 ： warm pools (min_workers=1)

最简单 mitigation: keep one replica always ready. 成本 is one GPU's hourly rate 24x7. The arithmetic is brutal on small 模型 (you pay $0.85-$1.50/hr to avoid a 30s 冷启动) and kind to large ones (pay $4/hr to avoid a 5-minute 冷启动). The SLA threshold where warm pools become mandatory: typically TTFT P99 < 60s on a 70B+ 模型.

### Layer 5 ： tiered loading (ServerlessLLM)

ServerlessLLM treats storage as a hierarchy: NVMe (fast but big), DRAM (medium but tiered), HBM (tiny but instant). Weights are pre-loaded to DRAM; load-按需 into HBM. Paper reports 10-200x 延迟 reduction on cold loads versus naive disk-to-HBM. 生产化 adoption is early but integrations with vLLM exist.

### Layer 6 ： live 迁移 (bonus 模式)

当a node becomes unavailable (spot eviction, node drain), traditional 模式 is cold-start another replica and drain 请求 队列. Live 迁移 moves 这个输入 词元 (kilobytes) to a destination that has 这个模型 loaded and recomputes KV 缓存 on the destination. Recomputation is 更便宜 than transferring GB of KV 缓存 over the network. Applicable to disaggregated deployments.

### The warm-pool math

For a service with P99 TTFT SLA of 2s, the question is not "预热池 yes/no" but "how many warm replicas, and which paths get them."

- High-value interactive paths (live chat, voice agent): `min_workers=1-2`.
- Background 批处理 paths (nightly classification): scale-to-zero accepted, 5-10 minute 冷启动 tolerable.
- Premium tier: `min_workers` per tenant with 专用容量.

### 先测量，再优化

Cold-start anatomy for a 70B 模型 on a fresh node (illustrative):

| Phase | Time | Mitigation |
|-------|------|-----------|
| Node provision | 50s | Bottlerocket + pre-seeded image, 预热池 |
| Image pull | 180s | Pre-seeded data volume (eliminate) |
| Weights to HBM | 75s | 模型 streamer (halve); GPU snapshot (eliminate) |
| Engine init | 20s | Persistent CUDA graph cache |
| First forward | 3s | Min inherent 延迟 |
| **Total cold** | **328s** | |
| **Total with mitigations** | **~15s** | 22x reduction |

### 你应该记住的数字

- Modal 冷启动: 2-4s (with GPU snapshots).
- Baseten 默认 冷启动: 5-10s; sub-second with pre-warming.
- Raw 70B 冷启动: 3-8 minutes.
- Run:ai 模型 Streamer: ~2x weight-load speedup.
- ServerlessLLM tiered loading: 10-200x 延迟 reduction (paper numbers).

## 使用它

`code/main.py` 模型 a cold-start path with and without each mitigation. Reports total cold-start time, warm-pool 成本, and 这个盈亏平衡 请求 rate above which 预热池 pays for itself.

## 交付它

This lesson 产出 `outputs/skill-cold-start-planner.md`. Given SLA, 模型 size, 和 流量 shape, picks which mitigations to stack.

## 练习

1. Run `code/main.py`. Compute 这个盈亏平衡 请求 rate above which a warm replica is 更便宜 than paying the cold-start tax via extra 请求 drops at SLO.
2. You deploy a 13B 模型 with P99 TTFT SLA of 3s. Pick the minimum mitigation stack (fewest layers) that achieves it.
3. Bottlerocket pre-seeding eliminates image pull but weights still load from snapshot to HBM. Compute wall-clock for a 70B 模型 if the snapshot-backed NVMe reads at 7 GB/s.
4. Your Serverless 提供商 offers GPU snapshots (Modal) and your 团队 refuses because "snapshots leak PII." Argue both sides ： what is the realistic 风险, and what is the mitigation (ephemeral snapshots, encryption, namespace isolation)?
5. Design a tiered warm-pool 策略: how many warm replicas for paid users, trial users, 和 批处理 工作负载? Show the math.

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| 冷启动 | "the big pause" | Time from 请求 to first 词元 on a fresh replica |
| 预热池 | "always-on minimum" | `min_workers >= 1` to keep at least one replica ready |
| Pre-seeded image | "baked AMI" | Node image with container weights pre-resident |
| Bottlerocket | "AWS node OS" | AWS container-optimized OS with dual-volume snapshot support |
| 模型 streamer | "streaming load" | Overlap weights I/O with compute setup |
| GPU snapshot | "checkpoint to HBM" | Serialize post-load GPU state; deserialize on restart |
| Tiered loading | "NVMe + DRAM + HBM" | Hierarchy of storage tiers; load on demand |
| Live 迁移 | "move 词元" | Transfer 输入 (KB), recompute KV on destination |
| `min_workers` | "warm replicas" | Serverless minimum keep-alive count |
| Scale-to-zero | "full Serverless" | No 成本 when 空闲; accept full cold-start tax |

## 延伸阅读

- [Modal — Cold start performance](https://modal.com/docs/guide/cold-start) ： Modal's published 基准 and checkpoint architecture.
- [AWS Bottlerocket](https://github.com/bottlerocket-os/bottlerocket) ： pre-seeded data volume snapshot 模式.
- [NVIDIA Run:ai Model Streamer](https://github.com/run-ai/runai-model-streamer) ： overlap weights load with compute setup.
- [Baseten — Cold-start mitigation](https://www.baseten.co/blog/cold-start-mitigation/) ： pre-warming playbook.
- [ServerlessLLM paper (USENIX OSDI'24)](https://www.usenix.org/conference/osdi24/presentation/fu) ： tiered loading design.
- [NVIDIA — Disaggregated LLM Inference on Kubernetes](https://developer.nvidia.com/blog/deploying-disaggregated-llm-inference-workloads-on-kubernetes/) ： live 迁移 for disaggregated deployments.
