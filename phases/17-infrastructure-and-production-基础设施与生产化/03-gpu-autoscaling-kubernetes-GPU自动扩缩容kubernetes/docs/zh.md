# Kubernetes 上的 GPU 自动扩缩容：Karpenter、KAI Scheduler、Gang Scheduling

> Three layers, not one. Karpenter provisions nodes dynamically (under one minute, 40% faster than Cluster Autoscaler). KAI 调度器 handles Gang Scheduling, topology awareness, and hierarchical queues ： it prevents the 7-of-8 partial allocation trap where seven nodes wait and burn on one missing GPU. Application-level autoscalers (NVIDIA Dynamo Planner, llm-d 工作负载 Variant Autoscaler) scale on inference-具体 signals ： 队列 depth, KV 缓存 utilization ： not CPU/DCGM duty cycle. The classic HPA trap is that `DCGM_FI_DEV_GPU_UTIL` is a duty-cycle measurement: 100% could be 10 请求 or 100. vLLM pre-allocates KV 缓存 内存, so 内存 never triggers scale-down. This lesson teaches you to compose the three layers and avoid 这个默认 Karpenter `WhenEmptyOrUnderutilized` 策略 that terminates running GPU jobs mid-inference.

**Type:** Learn
**Languages:** Python（标准库， 玩具 queue-depth autoscaler模拟器)
**Prerequisites:** Phase 17 · 02 (Inference Platform Economics), Phase 17 · 04 (vLLM Serving Internals)
**Time:** ~75 分钟

## 学习目标

- Diagram the three 自动扩缩容 layers (节点供应, Gang Scheduling, application-level) and name the tool used at each layer.
- 解释 为什么 `DCGM_FI_DEV_GPU_UTIL` is the wrong HPA signal for vLLM and name two replacements (队列 depth, KV 缓存 utilization).
- Describe Gang Scheduling and the partial-allocation failure mode KAI 调度器 prevents (7 of 8 GPUs 空闲).
- Name the Karpenter consolidation 策略 (`WhenEmptyOrUnderutilized`) that terminates running GPU jobs and state the 2026 safe alternative.

## 问题

Your 团队 ships an LLM-serving service on Kubernetes. You set up HPA with `DCGM_FI_DEV_GPU_UTIL` as the signal. The service pins at 100% utilization during business hours. HPA never scales up ： it already thinks you're full. You add a replica manually; TTFT drops. HPA still doesn't scale. The signal is lying to you.

Separately, you use Cluster Autoscaler for nodes. A 1M-词元 提示词 arrives at 2 a.m.; the cluster spends 3 minutes provisioning a node, and 这个请求 times out.

Separately again, you deploy a 70B 模型 requiring 8 GPUs across 2 nodes. The cluster has 7 GPUs free and 1 spread across 3 nodes. Cluster Autoscaler provisions a node for the 1 missing GPU. Seven nodes wait 4 minutes burning money while Kubernetes gets the last GPU up.

Three layers, three different failure modes. GPU-aware 自动扩缩容 in 2026 is not "turn on HPA." It's composing 节点供应, Gang Scheduling, and application-signal 自动扩缩容.

## 概念

### 第一层：节点供应（Karpenter）

Karpenter watches pending pods and provisions nodes within ~45-60 seconds (Cluster Autoscaler typically takes 90-120 seconds for GPU nodes). It picks instance types dynamically per 这个`NodePool` constraint ： if your pod needs 8 H100s and the cluster has no matching node, Karpenter provisions one 直接 instead of scaling an existing group.

**The consolidation trap**: Karpenter's 默认 `consolidationPolicy: WhenEmptyOrUnderutilized` is dangerous for GPU pools. It will terminate a running GPU node to migrate pods to 一个更便宜 right-sized instance. For inference 工作负载 that means evicting running 请求 and reloading a 70B 模型 on the new node. Loss is minutes of 容量 plus 请求 failures.

Safe setting for GPU pools:

```yaml
disruption:
  consolidationPolicy: WhenEmpty
  consolidateAfter: 1h
```

Lets Karpenter consolidate truly empty nodes after 一个小时 but never evict a running job.

### 第二层：Gang Scheduling（KAI Scheduler）

KAI 调度器 (project "Karp" then renamed) handles what 默认 kube-调度器 does not:

**Gang Scheduling** ： schedule all-or-nothing. A distributed inference pod requiring 8 GPUs either all 8 start together or none do. Without this, you get the partial-allocation trap: 7 of 8 pods start, wait indefinitely, burn money.

**Topology awareness** ： know which GPUs share NVLink, which sit on the same rack, which have InfiniBand between them. Place pods accordingly. A DeepSeek-V3 67B tensor-parallel 工作负载 must stay on one NVLink domain; KAI 调度器 respects that.

**Hierarchical queues** ： multiple 团队 compete for the same GPU pool with priority and quota. 团队 A's 生产化 pinch gets preempted by 团队 B's training job only if priority rules allow.

KAI is deployed alongside kube-调度器 as 一个次要调度器; you annotate 工作负载 to use it. Ray and vLLM 生产化-stack both integrate.

### 第三层：应用级信号

**The HPA trap**: `DCGM_FI_DEV_GPU_UTIL` is a duty-cycle 指标 ： it measures whether the GPU was doing work at each sampling interval. 100% utilization could 均值 10 concurrent 请求 or 100; the GPU was busy either way. Scaling on duty cycle is scaling blindly.

Worse, vLLM and 类似 engines pre-allocate KV 缓存 内存 (up to `--gpu-memory-utilization`). 内存 usage stays near 90% even at one 请求. 内存-based HPA never scales down.

**2026 replacement signals**:

- 队列 depth (number of 请求 waiting for prefill).
- KV 缓存 utilization (what fraction of blocks are allocated to active sequences).
- Per-replica P99 TTFT (your SLA signal).
- Goodput (请求 meeting all SLOs per second).

NVIDIA Dynamo Planner and llm-d 工作负载 Variant Autoscaler consume these signals and scale replicas. They replace HPA entirely for LLM serving.

### 何时使用哪种方案

| Scale 决策 | Tool |
|----------------|------|
| Add/remove nodes | Karpenter |
| Schedule multi-GPU jobs | KAI 调度器 |
| Add/remove replicas | Dynamo Planner / llm-d WVA (or custom HPA on 队列 depth) |
| Choose GPU type | Karpenter NodePool |
| Preempt low-priority | KAI 调度器 queues |

### 预填充/解码分离让一切更复杂

如果you run disaggregated prefill/decode (Phase 17 · 17), you have two pod classes with different scaling triggers: prefill pods scale on 队列 depth, decode pods scale on KV 缓存 pressure. llm-d exposes these as separate `Services` with per-role HPA. Do not try to put a single HPA in front of both.

### 冷启动在这里同样重要

Cold-start mitigation (Phase 17 · 10) is where 节点供应 time becomes user-visible. Karpenter's 45-60 second warm-up plus a 20GB 模型 load plus engine init means a from-zero 请求 takes 2-5 minutes. Keep 一个预热池 (`min_workers=1`) for SLO-关键 paths, or use Modal-style checkpointing at application layer.

### 你应该记住的数字

- Karpenter 节点供应: ~45-60s 对比 Cluster Autoscaler ~90-120s (GPU nodes).
- KAI 调度器 prevents partial-allocation waste ： 7-of-8 trap.
- `DCGM_FI_DEV_GPU_UTIL` as HPA signal: broken; use 队列 depth or KV utilization.
- Karpenter `WhenEmptyOrUnderutilized`: terminates running GPU jobs. Use `WhenEmpty + consolidateAfter: 1h` for inference.

```figure
autoscaling
```

## 使用它

`code/main.py` simulates a three-layer autoscaler on a bursty GPU 工作负载. Compares naive HPA (duty cycle), 队列-depth HPA, and KAI-gang-scheduled scaling. Reports unmet 请求, 空闲-GPU minutes, and a composite score.

## 交付它

This lesson 产出 `outputs/skill-gpu-autoscaler-plan.md`. Given cluster topology, 工作负载 shape, and SLO, it designs a three-layer 自动扩缩容 计划.

## 练习

1. Run `code/main.py`. Under a bursty 工作负载, how many 请求 does naive duty-cycle HPA drop that 队列-depth HPA catches? Where does the difference come from?
2. Design a Karpenter NodePool for a cluster serving Llama 3.3 70B FP8 on H100 SXM5. Specify `capacity-type`, `disruption.consolidationPolicy`, `consolidateAfter`, and a taint that keeps non-GPU 工作负载 off these nodes.
3. Your 团队 reports that deployments are stuck in Pending because "GPUs 可用 but pod won't schedule." Diagnose ： is this Karpenter, kube-调度器, or KAI 调度器? Which 指标 confirm?
4. Pick a signal to autoscale disaggregated prefill pods and a different signal for decode pods. 说明理由 both.
5. Compute 这个成本 of 这个`WhenEmptyOrUnderutilized` consolidation trap on a 24x7 生产化 service that averages 60 请求-dropping events/day at P99 TTFT > 10s.

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| Karpenter | "the node provisioner" | Kubernetes node autoscaler; sub-minute provisioning |
| Cluster Autoscaler | "the old scaler" | Kubernetes node autoscaler predecessor; slower, group-based |
| KAI 调度器 | "the GPU 调度器" | 次要调度器 for gang + topology + queues |
| Gang Scheduling | "all or nothing" | Schedule N pods atomically or defer all of them |
| Topology awareness | "rack-aware" | Place pods based on NVLink/IB/rack placement |
| `DCGM_FI_DEV_GPU_UTIL` | "GPU utilization" | Duty-cycle 指标; NOT a scaling signal for LLMs |
| 队列 depth | "waiting 请求" | Correct HPA signal for prefill-bound scaling |
| KV 缓存 utilization | "内存 pressure" | Correct HPA signal for decode-bound scaling |
| Consolidation | "Karpenter consolidation" | Node termination to 更便宜 instance type |
| `WhenEmpty + 1h` | "safe consolidation" | 策略 that doesn't evict running GPU jobs |

## 延伸阅读

- [KAI Scheduler GitHub](https://github.com/kai-scheduler/KAI-Scheduler) ： design docs and configuration examples.
- [Karpenter Disruption Controls](https://karpenter.sh/docs/concepts/disruption/) ： consolidation 策略 semantics and GPU-safe defaults.
- [NVIDIA — Disaggregated LLM Inference on Kubernetes](https://developer.nvidia.com/blog/deploying-disaggregated-llm-inference-workloads-on-kubernetes/) ： Dynamo Planner scaling signals.
- [Ray docs — KAI Scheduler for RayClusters](https://docs.ray.io/en/latest/cluster/kubernetes/k8s-ecosystem/kai-scheduler.html) ： Ray integration 模式.
- [AWS EKS Compute and Autoscaling Best Practices](https://docs.aws.amazon.com/eks/latest/best-practices/aiml-compute.html) ： managed-Kubernetes-具体 guidance.
- [llm-d GitHub](https://github.com/llm-d/llm-d) ： 工作负载 Variant Autoscaler design.
