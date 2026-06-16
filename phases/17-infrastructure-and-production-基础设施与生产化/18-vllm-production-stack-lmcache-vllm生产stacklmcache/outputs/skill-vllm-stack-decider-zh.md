---
name: vllm-stack-decider-zh
description: Decide vLLM 部署 layout ： 生产化-stack Helm chart, KV offload (native CPU or LMCache), 路由器/可观测性 integration ： 给定工作负载 and fleet size.
version: 1.0.0
phase: 17
lesson: 18
tags: [vllm, production-stack, lmcache, kv-offload, connector-api]
---

给定工作负载 (提示词 shape, concurrency, prefix reuse 模式), fleet (engines, GPU type), and operational context (Kubernetes-native, 多租户, budget), produce a vLLM stack 计划.

产出：

1. Stack. Use vLLM 生产化-stack Helm chart (recommended for new deployments) or roll your own. State which operators/CRDs apply.
2. KV offload. Choose:
   - None (short 提示词, low concurrency ： overhead exceeds benefit).
   - Native vLLM CPU offload (single-engine HBM pressure, 简单).
   - LMCache connector (multi-engine prefix reuse, preemption-heavy, 或 多租户 shared 提示词).
3. HBM utilization 监控. Set `--gpu-memory-utilization` with headroom; alert at 92%+ sustained as a pre-preemption signal.
4. 路由器 integration. Cache-aware 路由器 (Phase 17 · 11). Confirm KV-event channel configured.
5. 可观测性. Prometheus scrape per engine, OTel GenAI attributes (Phase 17 · 13), Grafana dashboard template from 生产化-stack.
6. Expected impact. Quantify expected 吞吐量 gain 对比 current ： reference the 16x H100 基准 shape (LMCache helps when KV footprint exceeds HBM).

硬性拒绝：
- Deploying LMCache without shared prefixes or preemption. 拒绝 ： overhead, no benefit.
- Running vLLM without HBM-pressure 监控. 拒绝 ： first preemption will be a surprise.
- Hand-rolling 生产化-stack when the Helm chart covers the use case. 拒绝 ： reinvent 成本.

拒绝规则：
- If the fleet has <2 engines, 拒绝 LMCache ： cross-engine reuse is the point; single-engine use native.
- If 这个工作负载 has 提示词 < 1K 词元 和 < 100 concurrency, 拒绝 offload of any kind ： HBM headroom suffices.
- If 这个团队 doesn't have K8s capability, 拒绝 生产化-stack ： start with a single-engine vLLM + 简单 proxy.

输出： a one-page 计划 naming stack, KV offload choice, HBM 监控, 路由器 integration, 可观测性, expected impact. End with the single gate: HBM utilization P99 over last 24h.
