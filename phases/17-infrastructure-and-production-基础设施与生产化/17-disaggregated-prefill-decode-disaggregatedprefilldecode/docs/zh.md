# 预填充/解码分离：NVIDIA Dynamo 与 llm-d

> Prefill is compute-bound; decode is 内存-bound. Running both on the same GPU wastes one resource. Disaggregation splits them onto separate pools and transfers KV 缓存 between them over NIXL (RDMA/InfiniBand or TCP fallback). NVIDIA Dynamo (GTC 2025 announce, 1.0 GA) sits above vLLM/SGLang/TRT-LLM ： its Planner Profiler + SLA Planner auto-rate-match prefill:decode ratios to meet SLOs. NVIDIA publishes 吞吐量 gains in this ballpark ： developer.nvidia.com (2025-06) shows 一个~6x improvement for DeepSeek-R1 MoE on GB200 NVL72 + Dynamo in the medium-延迟 regime, and the Dynamo 产品 page (developer.nvidia.com, undated) advertises up to 50x MoE 吞吐量 on GB300 NVL72 + Dynamo 对比 Hopper. 这个"30x" figure is a community aggregate across full-stack Blackwell + Dynamo + DeepSeek-R1 reports; we have not found a single primary source stating exactly 30x, so treat it as a directional claim. llm-d (Red Hat + AWS) is Kubernetes-native: prefill / decode / 路由器 as independent Services with per-role HPA. llm-d 0.5 adds hierarchical KV offloading, cache-aware LoRA 路由, UCCL networking, scale-to-zero. Economics: internal rollup of multiple 客户 disclosures suggests 30–40% 节省 on $2M-class inference spend (i.e., $600-800K/year) when switching from colocated serving to disaggregated with Dynamo at constant SLA; 这个具体$2M→$600-800K figure is an internal composite, not a single published case study ： use it as an order-of-magnitude anchor, not a reference citation. Short 提示词 (<512 词元, short 输出) don't 说明理由 the transfer 成本.

**Type:** Learn
**Languages:** Python（标准库， 玩具 disaggregated-vs-colocated模拟器)
**Prerequisites:** Phase 17 · 04 (vLLM Serving Internals), Phase 17 · 08 (Inference Metrics)
**Time:** ~75 分钟

## 学习目标

- 解释 why prefill and decode have different optimal GPU allocations and quantify the waste under colocation.
- Diagram the disaggregated architecture: prefill pool, decode pool, KV transfer via NIXL, 路由器.
- Name the condition when disaggregation does NOT pay off (short 提示词, short 输出).
- Distinguish NVIDIA Dynamo (stack-above) from llm-d (Kubernetes-native) and match each to an operational context.

## 问题

You run Llama 3.3 70B on 8 H100s. Under mixed 工作负载 (long 提示词 + short 输出), GPUs 空闲 during decode because most of the compute was spent on prefill. Under 不同工作负载 (short 提示词 + long 输出), the opposite happens. Colocated prefill + decode means you over-provision both.

Budget impact: 20-40% of GPU time is wasted on the wrong resource. You are buying H100 compute to run 内存-bound decode, or buying H100 HBM 带宽 to run compute-bound prefill. Both are expensive waste.

Disaggregation splits prefill and decode onto separate pools sized for each's bottleneck. KV 缓存 transfers from prefill pool to decode pool via high-带宽 interconnect.

## 概念

### 为什么瓶颈不同

**Prefill** ： run the transformer over the full 输入 提示词 in one forward. Matrix multiplications dominate; compute-bound. H100 FP8 gives ~2000 TFLOPS of useful 吞吐量. 批处理 efficiency is good ： one forward processes many 词元.

**Decode** ： generate one 词元 at a time, reading the full weights each iteration. 内存-带宽-bound. HBM3 gives ~3 TB/s. 批处理 efficiency is good only at high concurrency ： the weights read amortizes across 这个批处理.

Colocating them: you buy GPUs optimized for both. H100 is good at both but costs the same either way. At scale, you want prefill pool on H100 / compute-heavy; decode pool on H200 / 内存-heavy, or with aggressive 量化.

### 架构

```
            ┌──────────────┐
  Request → │    Router    │ ───────────────────────┐
            └──────┬───────┘                        │
                   │                                │
                   ▼ (prompt only)                  │
            ┌──────────────┐    KV cache    ┌───────▼──────┐
            │ Prefill pool │ ─── NIXL ────► │ Decode pool  │
            │  (compute)   │                │  (memory)    │
            └──────────────┘                └──────┬───────┘
                                                   │ tokens
                                                   ▼
                                                 Client
```

NIXL is NVIDIA's inter-node transport. Uses RDMA/InfiniBand when 可用, TCP fallback otherwise. Transfer 延迟 is real ： typically 20-80 ms for KV 缓存 of a 4K-词元 提示词 on 70B FP8. This is why short 提示词 don't 说明理由 disaggregation: the transfer tax exceeds 这个节省.

### Dynamo 对比 llm-d

**NVIDIA Dynamo** (GTC 2025 announce, 1.0 GA):
- Sits above vLLM, SGLang, TRT-LLM as an orchestrator.
- Planner Profiler measures 工作负载, SLA Planner auto-configures prefill:decode ratios.
- Rust core, Python extensibility.
- 吞吐量 gains: NVIDIA reports 6x for DeepSeek-R1 MoE on GB200 NVL72 + Dynamo in the medium-延迟 regime (developer.nvidia.com, 2025-06); community reports of "up to 30x" on full Blackwell + Dynamo + DeepSeek-R1 stacks lack a single primary source and should be treated as directional.
- GB300 NVL72 + Dynamo: up to 50x MoE 吞吐量 对比 Hopper per the Dynamo 产品 page (developer.nvidia.com, undated).

**llm-d** (Red Hat + AWS, Kubernetes-native):
- Prefill / decode / 路由器 as independent Kubernetes Services.
- Per-role HPA with 队列 depth (prefill) / KV utilization (decode) signals.
- `topologyConstraint packDomain: rack` packs prefill+decode cliques on the same rack for high-带宽 KV transfer.
- llm-d 0.5 (2026): hierarchical KV offloading, cache-aware LoRA 路由, UCCL networking, scale-to-zero.

使用Dynamo if you want a managed stack-above orchestrator. Use llm-d if you want Kubernetes-native primitives and are committed to the CNCF ecosystem.

### 经济学

Internal composite (not a single published case study ： order-of-magnitude anchor):

- $2M/year inference spend on colocated serving.
- Switched to disaggregated with Dynamo.
- Same 请求 volume, same P99 延迟 SLA.
- Reported 节省: $600K–$800K/year (30–40% reduction).
- No new hardware.

We synthesize this figure from multiple 客户 disclosures rather than a single citable case study; closest published data point is Baseten's 2x faster TTFT / 61% higher 吞吐量 with Dynamo KV 路由 (baseten.co, 2025-10), and VAST + CoreWeave's projection of 60–130% more 词元/$ at 40–60% KV hit rate (vastdata.com, 2025-12). 这个节省 come from right-sizing each pool; prefill-heavy 工作负载 (RAG with 8K+ prefixes) benefit more than balanced ones.

### 什么时候不要分离

- 提示词 < 512 词元 和 输出 < 200 词元: transfer tax dominates gain.
- Small cluster (< 4 GPUs): not enough pool diversity.
- 团队 cannot operate two GPU pools with per-role scaling: Dynamo helps but not trivially.
- No RDMA fabric: TCP transfer tax is heavier.

### 这个路由器 integrates with Phase 17 · 11

Disaggregated routers are KV-cache-aware (Phase 17 · 11). 一个请求 lands on the decode pool holding its prefix ： if no match, it flows prefill → decode. Hit rate and disaggregation compound ： the cache-aware 路由器 determines whether a new prefill is even 需要.

### MoE on Blackwell is where the real numbers are

GB300 NVL72 + Dynamo shows 50x MoE 吞吐量 over Hopper baselines. MoE expert 路由 is compute-heavy on prefill but 内存-heavy on decode (expert caches), so disaggregation is a double win. 2026 frontier 模型 serving is MoE-dominant (DeepSeek-V3, future GPT-5 variants).

### 你应该记住的数字

基准 numbers drift ： NVIDIA and the inference stack post updated results every quarter. Re-check before quoting.

- DeepSeek-R1 on GB200 NVL72 + Dynamo: ~6x 吞吐量 对比 baseline in the medium-延迟 regime (developer.nvidia.com, 2025-06); community "up to 30x" claims on full Blackwell + Dynamo stacks are directional aggregates without a single primary source.
- GB300 NVL72 + Dynamo: up to 50x MoE 吞吐量 对比 Hopper (developer.nvidia.com, undated).
- 节省 anchor (internal composite, not a single case study): $600-800K/year off 一个$2M annual spend at constant SLA.
- Disaggregation threshold: 提示词 >512 词元 + 输出 >200 词元.
- KV transfer via NIXL: 20-80 ms for 4K-提示词 KV on 70B FP8.

## 使用它

`code/main.py` simulates colocated 对比 disaggregated serving. Reports 吞吐量, 成本 per 请求, and 这个提示词-length crossover.

## 交付它

This lesson 产出 `outputs/skill-disaggregation-decider.md`. 给定工作负载 and cluster, decides whether to disaggregate.

## 练习

1. Run `code/main.py`. At what 提示词 length does disaggregation beat colocation?
2. Design the prefill pool and decode pool for a RAG service with P99 prefix length 8K, 输出 300.
3. Dynamo 对比 llm-d: pick one for a pure-Kubernetes shop with no Python runtime preference.
4. Compute KV transfer 成本: 4K prefill on 70B FP8 = ~500 MB KV. At RDMA 100 GB/s, transfer = 5 ms. At TCP 10 GB/s = 50 ms. Which matters for your SLA?
5. MoE expert 路由 changes KV access patterns. How does disaggregation behave with MoE that activates different experts per 词元?

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| Disaggregated serving | "split prefill/decode" | Separate GPU pools for each phase |
| NIXL | "NVIDIA transport" | Dynamo's inter-node KV transfer (RDMA/TCP) |
| NVIDIA Dynamo | "the orchestrator" | Stack-above coordinator for vLLM/SGLang/TRT-LLM |
| llm-d | "Kubernetes native" | Red Hat + AWS K8s disaggregated stack |
| Planner Profiler | "Dynamo auto-config" | Measures 工作负载, configures pool ratios |
| SLA Planner | "Dynamo 策略" | Auto-rate-matches prefill:decode to meet SLOs |
| `packDomain: rack` | "llm-d topology" | Pack prefill+decode on same rack for fast KV |
| UCCL | "统一 collective" | llm-d 0.5 networking layer for scale-to-zero |
| MoE expert 路由 | "expert per 词元" | DeepSeek-V3 模式; disaggregation helps |

## 延伸阅读

- [NVIDIA — Introducing Dynamo](https://developer.nvidia.com/blog/introducing-nvidia-dynamo-a-low-latency-distributed-inference-framework-for-scaling-reasoning-ai-models/)
- [NVIDIA — Disaggregated LLM Inference on Kubernetes](https://developer.nvidia.com/blog/deploying-disaggregated-llm-inference-workloads-on-kubernetes/)
- [TensorRT-LLM Disaggregated Serving blog](https://nvidia.github.io/TensorRT-LLM/blogs/tech_blog/blog5_Disaggregated_Serving_in_TensorRT-LLM.html)
- [llm-d GitHub](https://github.com/llm-d/llm-d)
- [llm-d 0.5 release notes](https://github.com/llm-d/llm-d/releases)
