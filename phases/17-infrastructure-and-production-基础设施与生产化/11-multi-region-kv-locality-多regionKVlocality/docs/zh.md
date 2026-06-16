# 多区域大模型服务与 KV 缓存局部性

> Round-robin load balancing is actively harmful for cached LLM inference. 一个请求 that does not land on the node holding its prefix pays full prefill 成本 ： roughly 800 ms at P50 on a long 提示词 versus ~80 ms with a cache hit. In 2026 这个生产化 模式 is a cache-aware 路由器 (vLLM 路由器 in Rust, llm-d 路由器) that consumes KV-cache events and routes on prefix-hash match. Recent research (GORGO) makes cross-区域 network 延迟 an explicit term in 这个路由 objective. Commercial "cross-区域 inference" offerings (Bedrock cross-区域 inference, GKE multi-cluster gateways) treat inference as 不透明 ： they handle availability, not TTFT. JPMorgan and Mayo Clinic ran us-east-1 故障切换 in Nov 2024 at ~22 minutes. The DR reality: 32% of LLM DR failures are because 团队 backed up weights but forgot tokenizer files or 量化 configs.

**Type:** Learn
**Languages:** Python（标准库， 玩具 prefix-cache-aware router模拟器)
**Prerequisites:** Phase 17 · 04 (vLLM Serving), Phase 17 · 06 (SGLang RadixAttention)
**Time:** ~60 分钟

## 学习目标

- 解释 why round-robin load balancing breaks cached inference and quantify the TTFT penalty.
- Diagram a cache-aware 路由器: 输入 (KV-cache events), algorithm (prefix-hash match), tie-breaker (GPU utilization).
- Name the 32% DR failure driver for LLMs (missing tokenizer files / 量化 configs) and state a three-file DR checklist.
- Distinguish commercial cross-区域 offerings (Bedrock CRI, GKE Multi-Cluster 网关) from KV-aware 路由.

## 问题

Your service runs in us-east-1, us-west-2, and eu-west-1. You put an ALB in front with round-robin. 前缀缓存 hit rate in 生产化 drops to 8%. TTFT P50 triples. Your vLLM 日志 show every 请求 is paying full prefill 成本.

Round-robin is optimal for stateless services. LLM inference is stateful by design ： the KV 缓存 encodes everything 这个模型 has seen. 路由 blind is 路由 into the wrong cache.

Separately, your 团队 has a DR 计划. You back up 模型 weights to S3 cross-区域. A regional outage hits; you attempt 故障切换; the replica refuses to start. You forgot tokenizer.json, 这个量化 config, and the RoPE scaling config were in a separate bucket you didn't sync.

Multi-区域 LLM serving is a cache problem, 一个路由 problem, and a DR-hygiene problem ： not a load-balancer problem.

## 概念

### 缓存感知路由

请求 arrives with 一个提示词. 路由器 hashes the prefix (say, first 512 词元); it asks each replica "do you have this prefix cached?". Replicas publish KV-cache events on a pub/sub channel as they allocate and evict blocks. 路由器 picks the replica with the match, falls through to GPU-util-based tie-breaker if no one does.

**vLLM 路由器** (Rust, 2026 生产化-stack): subscribes to `kv.cache.block_added` events, maintains a prefix-hash → replica index, routes with O(1) lookup. Falls through to least-队列-depth when no match.

**llm-d 路由器**: same 模式, Kubernetes-native. Publishes events via the ControlPlane API.

**SGLang RadixAttention** (Phase 17 · 06) is the intra-replica equivalent. Cross-replica 路由 is strictly upstream.

### Numbers

TTFT P50 on a 2K-词元 提示词, Llama 3.3 70B FP8, H100:
- Cache hit (same replica, prefix resident): ~80 ms.
- Cache miss (cold prefill): ~800 ms.

10x gap. If your 路由器 hits 60-80% of 前缀缓存 across replicas, you approximate single-replica performance at N-replica 容量. If it hits 10%, you approximate naive scaling.

### 跨区域有新的约束：网络延迟

Inter-区域 RTT:
- us-east-1 ↔ us-west-2: ~65 ms.
- us-east-1 ↔ eu-west-1: ~75 ms.
- us-east-1 ↔ ap-southeast-1: ~220 ms.

如果路由 takes 一个请求 from us-east-1 to a hot prefix in ap-southeast-1, the saved prefill (800 → 80 ms) is dwarfed by 440 ms round-trip. GORGO (2026 research) makes this explicit ： minimize `prefill_time + network_latency` jointly, not prefill alone. Often the answer is to keep 路由 regional except on massive multi-MB prefixes where prefill dominates.

### Commercial "cross-区域 inference" does not help here

AWS Bedrock cross-区域 inference automatically routes 请求 to other 区域 during 容量 pressure. It optimizes availability, not TTFT, and treats inference as 不透明. GKE Multi-Cluster 网关 is the same ： service-level 故障切换, no awareness of KV 缓存.

You still need an app-layer cache-aware 路由器 even when using these. They handle 这个"us-east-1 is on fire" case. Cache-aware 路由 handles the TTFT case.

### DR hygiene ： the 32% missing-files problem

Widely cited 2026 stat: 32% of LLM DR failures happen because 团队 backed up weights but forgot:

- `tokenizer.json` 或 `tokenizer.model`
- 量化 configs (`quantize_config.json`, AWQ scales, GPTQ zero-points)
- 模型-具体 configs (RoPE scaling, attention masks, chat templates)
- Engine config (`vllm_config.yaml`, sampling defaults, LoRA adapter manifests)

The fix is a three-file minimum DR manifest:

1. All files under the HF 模型 repo (weights + configs + tokenizer).
2. Engine-具体 serving config.
3. 部署 manifest (K8s YAML, Dockerfile, dependency lock).

Plus: run a DR drill quarterly. The JPMorgan us-east-1 drill hit 22 minutes recovery in Nov 2024 only because the playbook was rehearsed.

### 数据驻留是正交问题

EU 客户 PHI cannot leave EU. If your cache-aware 路由器 sends a Paris-originated 请求 to us-east-1 for a prefix match, you have violated GDPR regardless of TTFT gain. Partition routers by residency boundary before optimizing for cache.

### 你应该记住的数字

- Cache hit 对比 miss TTFT gap: ~10x (80 ms 对比 800 ms on 2K 提示词).
- Inter-区域 RTT US-EU: ~75 ms.
- DR failure: 32% miss tokenizer/quant configs.
- JPMorgan us-east-1 故障切换 Nov 2024: 22 minutes (30-min SLA).

## 使用它

`code/main.py` simulates three 路由 strategies (round-robin, cache-aware regional, cache-aware global) on a multi-区域 工作负载. Reports cache hit rate, TTFT P50/P99, and cross-区域 bill.

## 交付它

This lesson 产出 `outputs/skill-multi-region-router.md`. 给定区域, residency 约束, and SLA, designs 一个路由 计划.

## 练习

1. Run `code/main.py`. At what 提示词 length does cross-区域 路由 beat local-only 路由, given 75 ms RTT?
2. Your cache hit rate drops from 70% to 12%. Diagnose three possible causes and the observables that would confirm each.
3. Design a DR manifest for a 70B AWQ-quantized 模型 served in vLLM with 5 LoRA adapters. List every file and config.
4. Argue whether Bedrock cross-区域 inference is "enough" for a fintech with strict TTFT SLOs. Cite 具体 behaviors.
5. A Paris-origin 请求 matches a prefix in us-east-1. Do you 路由 it? Write 这个策略.

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| Cache-aware 路由 | "smart LB" | 路由 on prefix-hash match to KV-cache-holding replica |
| KV-cache events | "cache pub-sub" | Replicas publish block add/evict; 路由器 indexes |
| Prefix hash | "cache key" | Hash of first N 词元 used as 路由器 lookup |
| GORGO | "cross-区域 路由 research" | arXiv 2602.11688; network 延迟 as explicit term |
| Cross-区域 inference | "Bedrock CRI" | AWS 产品; availability 故障切换, not TTFT awareness |
| DR manifest | "the backup list" | Every file 需要 to restore ： not just weights |
| 数据驻留 | "GDPR boundary" | Legal constraint on which 区域 sees user data |
| RTT | "round-trip time" | Network 延迟; 75 ms US-EU, 220 ms US-APAC |
| LLM-aware LB | "cache-hit LB" | Cache-aware 路由器 as 一个产品 category |

## 延伸阅读

- [BentoML — Multi-cloud and cross-region inference](https://bentoml.com/llm/infrastructure-and-operations/multi-cloud-and-cross-region-inference)
- [arXiv — GORGO (2602.11688)](https://arxiv.org/html/2602.11688v1) ： cross-区域 KV-cache reuse with network 延迟 term.
- [TianPan — Multi-Region LLM Serving Cache Locality](https://tianpan.co/blog/2026-04-17-multi-region-llm-serving-data-residency-routing)
- [AWS Bedrock Cross-Region Inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html) ： availability 故障切换 documentation.
- [vLLM Production Stack Router](https://github.com/vllm-project/production-stack) ： cache-aware 路由器 source.
