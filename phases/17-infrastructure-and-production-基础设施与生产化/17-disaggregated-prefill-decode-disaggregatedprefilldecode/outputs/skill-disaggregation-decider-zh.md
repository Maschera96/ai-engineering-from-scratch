---
name: disaggregation-decider-zh
description: Decide whether to 采用 disaggregated prefill/decode (Dynamo or llm-d) for 一个给定工作负载 and cluster. Quantify prefill:decode ratios, KV transfer 成本, and the expected 节省.
version: 1.0.0
phase: 17
lesson: 17
tags: [disaggregated-serving, dynamo, llm-d, nixl, kv-transfer, prefill-decode]
---

给定工作负载 画像 (提示词/输出 length distribution, 模型, concurrency), cluster topology (GPUs, fabric, RDMA availability), and current serving 成本, produce a disaggregation 决策.

产出：

1. Disaggregate? Yes / No with numbered justification. Baseline: 提示词 > 512 AND 输出 > 200. Fabric: RDMA 可用 helps; TCP-only pushes 盈亏平衡 longer.
2. Stack choice. NVIDIA Dynamo (managed orchestrator above vLLM/SGLang/TRT-LLM) or llm-d (Kubernetes-native Services). Match to the operational context.
3. Prefill:decode ratio. Use Dynamo Planner Profiler readouts, or compute from 工作负载 shape (prefill TFLOPS 对比 decode bytes/sec). Example: 2 prefill : 1 decode for RAG-heavy; 1:2 for 输出-heavy.
4. KV transfer 计划. Named transport (NIXL over InfiniBand / RDMA / TCP fallback). Compute the per-请求 transfer tax for your 提示词 P99.
5. 路由器 integration. Cache-aware 路由器 (Phase 17 · 11) must be in front ： disaggregation without prefix matching loses the cache win.
6. Expected 节省. Compute 对比 colocated baseline; cite the published case (30-40% at same SLA).

硬性拒绝：
- Disaggregating short-提示词 工作负载 (<512 词元). 拒绝 ： the transfer tax dominates.
- Deploying without a cache-aware 路由器. 拒绝 ： blind 路由 negates the KV locality.
- Ignoring topology (rack packing). 拒绝 ： KV transfer over multi-rack hops costs more than RDMA on the same rack.

拒绝规则：
- If the cluster has < 4 GPUs, 拒绝 ： not enough pool diversity for disaggregation to pay off.
- If no RDMA/InfiniBand and no plans, note that TCP raises 这个盈亏平衡 to 提示词 >2K; re-evaluate.
- If 这个团队 cannot operate two GPU pools with per-role scaling, 拒绝 llm-d and require Dynamo as the managed alternative.

输出： a one-page 决策 with disaggregate Y/N, stack choice, ratio, transport, 路由器, expected 节省. End with the single 指标 to verify: KV transfer P99 延迟; gate on exceeding 一个计划-specified threshold.
