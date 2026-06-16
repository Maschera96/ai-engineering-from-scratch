# 带 LMCache KV 卸载的 vLLM 生产化栈

> vLLM's 生产化-stack is the reference Kubernetes 部署 ： 路由器, engines, 和 可观测性 wired together. LMCache is the KV-offloading layer that extracts KV 缓存 out of GPU 内存 and reuses it across queries and engines (CPU DRAM, then disk/Ceph). The vLLM 0.11.0 KV Offloading Connector (January 2026) makes this asynchronous and pluggable via the Connector API (v0.9.0+). Offload 延迟 is not user-facing. LMCache is valuable even without shared prefixes ： when a GPU runs out of KV slots, preempted 请求 can be restored from CPU instead of recomputing prefill. Published 基准 on 16x H100 (80GB HBM) across 4 a3-highgpu-4g: when KV 缓存 exceeds HBM, both native CPU offload and LMCache substantially improve 吞吐量; at low KV footprint, all configs match baseline with small overhead.

**Type:** Learn
**Languages:** Python（标准库， 玩具 KV-spill模拟器)
**Prerequisites:** Phase 17 · 04 (vLLM Serving Internals), Phase 17 · 06 (SGLang/RadixAttention)
**Time:** ~60 分钟

## 学习目标

- Diagram the vLLM 生产化-stack layers: 路由器, engines, KV offload, 可观测性.
- 解释 the KV Offloading Connector API (v0.9.0+) and how the 0.11.0 asynchronous path hides offload 延迟.
- Quantify when LMCache CPU-DRAM helps (KV > HBM) 对比 adds overhead (KV small enough to fit HBM).
- Pick between native vLLM CPU offload and LMCache connector 给定部署 约束.

## 问题

Your vLLM serving shows GPUs at 100% HBM with preemption events whenever concurrency climbs. 请求 get evicted, requeued, and you re-prefill the same 2K-词元 提示词 four times in a minute. GPU compute is spent on redundant prefills; goodput is well below raw 吞吐量.

Adding more GPUs costs linearly. Adding more HBM is not possible. But CPU DRAM is cheap ： one socket has 512 GB+ at 延迟 orders of magnitude worse than HBM but fine for "temporarily warm" KV 缓存.

LMCache extracts KV 缓存 to CPU DRAM so preempted 请求 recover fast, and repeated prefixes across engines share cache without each engine re-prefilling.

## 概念

### vLLM 生产化-stack

`github.com/vllm-project/production-stack` is the reference Kubernetes 部署:

- **路由器** ： cache-aware (Phase 17 · 11). Consumes KV events.
- **Engines** ： vLLM workers. One per GPU or per TP/PP group.
- **KV 缓存 offload** ： LMCache 部署 or native connector.
- **可观测性** ： Prometheus scrape, Grafana dashboards, OTel 追踪.
- **Control plane** ： service discovery, config, rolling updates.

Shipped as Helm chart + operator.

### The KV Offloading Connector API (v0.9.0+)

vLLM 0.9.0 introduced a Connector API for pluggable KV 缓存 backends. Your engine offloads blocks to the connector; connector stores them (RAM, disk, object storage, LMCache). 请求 needs a block, connector loads it back.

vLLM 0.11.0 (January 2026) adds an asynchronous offload path ： offload can happen in the background so the engine does not block on it in the common case. End-to-end 延迟 和 吞吐量 still depend on 工作负载 shape, KV 缓存 hit rate, and system pressure; vLLM's own notes call out that custom-kernel offload can degrade 吞吐量 at low hit rates and that async scheduling has known interaction issues with speculative decoding.

### Native CPU offload 对比 LMCache

**Native vLLM CPU offload**: engine-local. Stores KV blocks in host RAM. Fast to implement, zero network hop. Does not cross engines.

**LMCache connector**: cluster-scale. Stores blocks in a shared LMCache server (CPU DRAM + Ceph/S3 tier). Blocks are accessible to any engine. 16x H100 基准 published.

选择native when a single engine has HBM pressure. Pick LMCache when multiple engines share prefixes (RAG with common system 提示词, 多租户 with shared templates).

### 基准 behavior

The 16x H100 (80 GB HBM) spread across 4 a3-highgpu-4g test:

- Low KV footprint (short 提示词, low concurrency): all configs match baseline, LMCache adds ~3-5% overhead.
- Moderate footprint: LMCache starts to help on prefix reuse across engines.
- KV exceeds HBM: native CPU offload and LMCache both improve 吞吐量 substantially; LMCache larger gain because cross-engine sharing.

### 当LMCache is decisive

- 多租户 serving where system 提示词 are shared across tenants.
- RAG where document chunks repeat across queries.
- Fine-tuned variants (LoRA) on the same base where base-模型 KV reuse cuts redundant work.
- Preemption-heavy 工作负载: restore from CPU 更便宜 than re-prefill.

### 当NOT to enable

- Small HBM pressure ： you pay overhead without benefit.
- Short contexts (<1K 词元) ： transfer time > re-prefill.
- Single-tenant single-提示词 工作负载 ： no reuse to capture.

### Integration with disaggregated serving

Phase 17 · 17 disaggregated serving + LMCache compounds: KV transfers from prefill pool to decode pool land in LMCache if not used; subsequent queries pull from LMCache. Phase 17 · 11 cache-aware 路由器 can 路由 to the engine whose local OR LMCache-shared cache matches.

### 你应该记住的数字

- vLLM 0.9.0: Connector API shipped.
- vLLM 0.11.0 (Jan 2026): asynchronous offload path; end-to-end 延迟 impact depends on 工作负载, KV hit rate, and system pressure (not an absolute guarantee).
- 16x H100 基准: LMCache helps when KV footprint exceeds HBM.
- Small HBM pressure: 3-5% overhead without benefit.

```figure
zero-sharding
```

## 使用它

`code/main.py` simulates a preemption-heavy 工作负载 with and without LMCache. Reports re-prefills avoided, 吞吐量 gain, and 这个盈亏平衡 HBM utilization.

## 交付它

This lesson 产出 `outputs/skill-vllm-stack-decider.md`. 给定工作负载 shape and vLLM 部署, decides 原生对比 LMCache 对比 neither.

## 练习

1. Run `code/main.py`. At what HBM utilization does LMCache start paying?
2. A tenant shares a 6K-词元 system 提示词 across 200 queries/小时. Compute expected LMCache 节省 per tenant.
3. The LMCache server is a single point of failure. Design the HA strategy (replicas, fallback to native).
4. LMCache stores to Ceph on spinning disk. For a 4K-词元 KV at 70B FP8 (500 MB), what's the read time 对比 re-prefill?
5. Argue whether the vLLM 0.11.0 asynchronous path is "free" ： where does the overhead hide?

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| 生产化-stack | "the reference 部署" | vLLM's Kubernetes Helm chart + operator |
| Connector API | "KV backend interface" | vLLM 0.9.0+ pluggable KV store interface |
| Native CPU offload | "engine-local spill" | Store KV in host RAM of same engine |
| LMCache | "cluster KV 缓存" | Cross-engine KV 缓存 server on CPU DRAM + disk |
| 0.11.0 async | "non-blocking offload" | Offload hidden behind engine stream |
| Preemption | "evict to make room" | KV 缓存 shuffle when HBM full |
| Prefix reuse | "same system 提示词" | Multiple queries share beginning; cache hit |
| Ceph tier | "disk tier" | Durable storage below DRAM in the cache hierarchy |

## 延伸阅读

- [vLLM Blog — KV Offloading Connector (Jan 2026)](https://blog.vllm.ai/2026/01/08/kv-offloading-connector.html)
- [vLLM Production Stack GitHub](https://github.com/vllm-project/production-stack) ： Helm chart + operator.
- [LMCache for Enterprise-Scale LLM Inference (arXiv:2510.09665)](https://arxiv.org/html/2510.09665v2)
- [LMCache GitHub](https://github.com/LMCache/LMCache) ： Connector implementation.
- [vLLM 0.11.0 release notes](https://github.com/vllm-project/vllm/releases) ： asynchronous path details.
