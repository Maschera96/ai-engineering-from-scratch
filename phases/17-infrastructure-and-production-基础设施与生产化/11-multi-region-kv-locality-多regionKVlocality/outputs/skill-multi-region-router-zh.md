---
name: multi-region-router-zh
description: 设计a multi-区域 LLM 路由 计划 with KV-cache locality, residency boundaries, DR manifest, and a quarterly 故障切换 drill.
version: 1.0.0
phase: 17
lesson: 11
tags: [multi-region, kv-cache, routing, dr, bedrock-cri, vllm-router, llm-d, gorgo]
---

给定区域 in scope, residency boundaries, expected prefix-cache diversity, and TTFT SLA, produce a multi-区域 路由 and DR 计划.

产出：

1. 路由器 choice. Pick cache-aware 路由器 (vLLM 路由器, llm-d 路由器) and describe the KV-event channel. State the prefix-hash algorithm (e.g., 512-词元 rolling) and tie-breaker (least 队列 depth).
2. 路由 策略. Regional-first or global (GORGO-style) minimization of prefill + RTT? 说明理由 with 这个提示词-length distribution ： long 提示词 (>8K 词元) benefit from cross-区域 路由; short 提示词 do not.
3. Residency partitioning. Before any optimization: which 请求 are bound to which 区域 for legal reasons (GDPR, HIPAA). Forbid cross-residency 路由 even when TTFT improves.
4. Commercial CRI layer. Recommend whether to enable Bedrock Cross-区域 Inference or GKE Multi-Cluster 网关 as the availability layer. State clearly this layer is NOT a TTFT optimization.
5. DR manifest. Three-file minimum (HF repo + engine config + 部署 manifest). Verify tokenizer, 量化 configs, RoPE, chat templates, LoRA adapters are included. State the storage (S3 cross-区域 replication, multi-区域 GCS).
6. 故障切换 drill. Quarterly cadence. Who runs it, what gets measured (RTO, RPO, cache warm-up time). Target: 30-minute RTO matched to real 2024 JPMorgan drill.

硬性拒绝：
- Ignoring residency for 路由 optimization. 拒绝 ： GDPR violation beats TTFT gain.
- Claiming Bedrock CRI "solves" cross-区域 路由. 拒绝 ： CRI is availability, not TTFT.
- Backing up weights only. 拒绝 ： name the 32% DR failure statistic and require the three-file manifest.

拒绝规则：
- If only one 区域 is in scope, decline the 计划 ： single-区域 has different failure modes (Phase 17 · 03 covers it).
- If residency and TTFT SLA are incompatible (e.g., EU residency forcing prefill on cold prefix per 请求 with P99 TTFT < 100 ms on 8K 提示词), 拒绝 to 承诺 the SLA and escalate 这个产品 requirement.

输出： a one-page 计划 naming 路由器, 路由 策略, residency partitions, CRI layer posture, DR manifest, quarterly drill owner. End with the single 指标 to alert on: cross-区域 prefix-cache hit rate dropping below 一个计划-specified threshold.
