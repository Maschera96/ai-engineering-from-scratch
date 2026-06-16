---
name: inference-server-zh
description: 交付一个 speculative-decoding inference server，使用 EAGLE-3 或 P-EAGLE drafts、K8s autoscaling，并提供完整 throughput/latency/cost 报告。
version: 1.0.0
phase: 19
lesson: 14
tags: [capstone, inference, vllm, sglang, eagle-3, p-eagle, speculative-decoding, quantization, hpa]
---

给定两个开放目标模型（Llama 3.3 70B 和 Qwen3-Coder-30B MoE 或 GPT-OSS-120B），交付一个带 speculative decoding、quantization 和 Kubernetes autoscaling 的生产 serving stack。发布实测 speedups 和 tail-latency numbers。

构建计划：

1. 在 vLLM 0.7（或 SGLang 0.4）下部署目标模型，使用 FP8 Marlin quantization。
2. 从 Red Hat Speculators 加载 aligned EAGLE-3 draft（或通过 SpecForge 训练）。
3. Baseline numbers：无 speculation 时，batch 1/8/32 的 tokens/s 和 p50/p99 latency。
4. 启用 EAGLE-3。重新运行相同 benchmark。报告 speedup、acceptance rate、p99 tail-latency delta。
5. 启用 P-EAGLE parallel speculation；报告 deeper trees 何时有帮助、何时有害的拐点。
6. 跨 distributions 运行 benchmarks：ShareGPT、HumanEval、domain data。发布 acceptance-rate drift。
7. 在第二个目标模型（MoE）上重复；识别 routing-noise sensitivity 对 draft acceptance 的影响。
8. 部署到 Kubernetes，HPA 跟踪 `queue_wait_ms`。演示 load triples 时 scale-out。
9. 在匹配 evals 上对比 $/1M tokens 与 Anthropic Claude Sonnet 4.7 和 OpenAI GPT-5.4。

评估标准：

| Weight | Criterion | Measurement |
|:-:|---|---|
| 25 | 相比 baseline 的实测 speedup | 两个模型上在质量匹配时达到 2.5x+ throughput |
| 20 | 真实流量上的 acceptance rate | Per-distribution acceptance-rate report |
| 20 | P99 tail-latency 纪律 | batch 1/8/32 上有无 speculation 的 p99 |
| 20 | Ops | K8s deploy、HPA on queue-wait、smooth rollout、drain-first upgrade |
| 15 | Write-up 和 methodology | Metrics 推导清晰，baselines 匹配 |

硬性拒收：

- 只报告 steady-state throughput，而不报告 tail latency。
- HPA 基于 CPU 而不是 queue-wait。GPU 饱和时会 thrash。
- 忽略 draft-target version alignment。漂移 drafts 比不用 speculation 还贵。
- 成本对比遗漏 hosted APIs 的 prompt-caching discounts。

拒绝规则：

- 没有 rollout drain 时拒绝服务。请求进行中时原地升级直接淘汰。
- 拒绝报告跨 distributions 聚合的 acceptance rate。必须按 distribution 报告。
- 没有匹配 non-speculative number 时，拒绝声称 speculative-decoding 在 bs=32 获胜。

输出：一个仓库，包含 vLLM / SGLang configs、EAGLE-3 draft download script、K8s deployment manifests、基于 queue-wait 的 HPA config、ShareGPT / HumanEval / domain data benchmark harness、$/1M tokens comparison table，以及一份说明文档，列出 speculative decoding 引入的三种 tail-latency regressions 和对应缓解措施（batch gating、ngram fallback、quantization tweak）。
