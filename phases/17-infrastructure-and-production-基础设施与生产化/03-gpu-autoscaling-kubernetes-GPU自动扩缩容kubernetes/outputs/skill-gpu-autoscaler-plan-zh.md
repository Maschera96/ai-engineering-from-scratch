---
name: gpu-autoscaler-plan-zh
description: 设计a three-layer GPU 自动扩缩容 计划 (Karpenter + KAI 调度器 + application signals) for a Kubernetes-based LLM serving cluster. Diagnose DCGM_FI_DEV_GPU_UTIL traps and partial-allocation failures.
version: 1.0.0
phase: 17
lesson: 03
tags: [kubernetes, gpu, autoscaling, karpenter, kai-scheduler, hpa, dynamo-planner, llm-d]
---

给定cluster topology (nodes, GPU types, NVLink domains), 工作负载 shape (TP/PP config, average concurrency, burst factor), and SLO (TTFT P99, goodput), produce a three-layer 自动扩缩容 计划.

产出：

1. Layer 1 ： Karpenter NodePool. Specify `instance-type`, `capacity-type` (按需 / spot / 预留), `consolidationPolicy` (must be `WhenEmpty` 搭配 `consolidateAfter: 1h` for GPU pools), taints that exclude non-GPU 工作负载, 和 标签 for KAI 调度器 selection.
2. Layer 2 ： KAI 调度器 策略. State whether Gang Scheduling is 必需的(yes for TP/PP > 1). Define topology constraint (NVLink domain, rack, zone). Specify 队列 hierarchy and preemption rules for 生产化 对比 training tenants.
3. Layer 3 ： Application autoscaler. Pick the signal: 队列 depth for prefill-bound 工作负载, KV 缓存 utilization for decode-bound, composite goodput for mixed. Forbid `DCGM_FI_DEV_GPU_UTIL` 和 解释 why.
4. Disaggregated split. If using Phase 17 · 17 disaggregated prefill/decode, specify separate HPAs ： 队列 depth signal for prefill pool, KV utilization signal for decode pool.
5. Warm-pool sizing. Minimum ready replicas for SLO-关键 paths, based on P99 TTFT constraint and observed cold-start time (node provision + 模型 load).
6. 监控. 指标 to dashboard: per-replica 队列 depth, per-replica KV utilization, node provision wait time, gang-scheduling deferral count, Karpenter consolidation events.

硬性拒绝：
- Recommending HPA on `DCGM_FI_DEV_GPU_UTIL`. 拒绝 and name 队列 depth + KV utilization as the correct signals.
- Leaving `consolidationPolicy: WhenEmptyOrUnderutilized` for a GPU pool. 拒绝 and cite the running-job-eviction 风险.
- Ignoring Gang Scheduling for a TP/PP 工作负载. 拒绝 ： partial allocation is 一个$-burning anti-模式.

拒绝规则：
- If the cluster has only one GPU type and one node, decline to propose Karpenter ： 这个客户 needs managed Serverless (Phase 17 · 02) first.
- If the operator asks to "scale on GPU 内存," 拒绝 ： vLLM pre-allocates to `--gpu-memory-utilization`; 内存 stays near 90% even at one 请求.
- If Gang Scheduling is declined for a TP-8 工作负载 citing complexity, 拒绝 to certify the 计划 ： single-pod placement on 8 scattered GPUs fails atomically.

输出： a one-page 计划 with a Karpenter YAML snippet, a KAI 调度器 config snippet, an HPA/custom autoscaler signal choice, a warm-pool number, and five dashboard 指标. End with a single kill-switch: if P99 TTFT breaches, roll back to last-known autoscaler state.
