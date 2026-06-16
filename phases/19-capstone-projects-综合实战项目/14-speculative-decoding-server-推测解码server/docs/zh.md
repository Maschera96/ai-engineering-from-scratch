# Capstone 14 — 推测解码推理服务器

> vLLM 0.7 中的 EAGLE-3 在真实流量上提供了 2.5 到 3 倍吞吐。P-EAGLE，AWS 2026，进一步推进了并行推测。SGLang 的 SpecForge 支持大规模训练 draft heads。Red Hat 的 Speculators hub 为常见开源模型发布了对齐后的 drafts。TensorRT-LLM 把 speculative decoding 提升为 NVIDIA 生态中的一等公民。2026 年的生产 serving stack，通常是 vLLM 或 SGLang，配 EAGLE 家族 drafts、FP8 或 INT4 quantization，再用基于 queue-wait 的 HPA 自动扩缩容。这个 capstone 的目标是，服务两个开源模型，并达到基线 2.5 倍以上吞吐，同时给出完整的 tail-latency 报告。

**Type:** Capstone
**Languages:** Python (serving), C++ / CUDA (kernel inspection), YAML (configs)
**Prerequisites:** Phase 3 (deep learning), Phase 7 (transformers), Phase 10 (LLMs from scratch), Phase 17 (infrastructure)
**Phases exercised:** P3 · P7 · P10 · P17
**Time:** 30 hours

## Problem

到 2026 年，speculative decoding 已经成了基础能力。EAGLE-3 draft heads 在目标模型的 hidden states 上训练，一次预测前方 N 个 tokens。目标模型再在一次 pass 中完成验证。60% 到 80% 的 acceptance rates，足以把端到端吞吐提升到 2 到 3 倍。vLLM 0.7 已经原生集成了它。SGLang + SpecForge 提供训练流水线。Red Hat 的 Speculators 则直接发布了与 Llama 3.3 70B、Qwen3-Coder-30B MoE、GPT-OSS-120B 对齐的 drafts。

真正的技巧在 serving operations，而不在模型本身。Acceptance rate 会随着流量分布变化而漂移，比如 ShareGPT、代码流量和领域数据之间就会不同。在 rejection 情况下，tail latency 甚至可能比不用 speculation 更糟，所以你必须在多个 batch size 下报告 p99，而不是只给 steady-state tokens/sec。每 1M tokens 的成本，拿来对比 Anthropic / OpenAI API，是这个系统是否可信的关键杠杆。

## Concept

Speculative decoding 有两层。**Draft** 模型，可以是 EAGLE-3 head、ngram，或者更小的目标对齐模型，在每一步提出 k 个候选 tokens。**Target** 模型会在一次 pass 中验证全部 k 个 tokens。任何被接受的前缀都会替代原本的 greedy path。Acceptance rate 取决于 draft 和 target 的对齐程度，以及输入流量的分布。

在大多数流量上，EAGLE-3 都优于 ngram drafts。P-EAGLE 则用并行推测支持更深的 draft tree。代价是，一旦 rejection 发生，P99 latency 会升高，因为 verify pass 更大。因此 serving config 必须按 batch-size bucket 细分报告 latency，把这个问题显露出来。

部署目标是 Kubernetes。vLLM 0.7 会按每个 GPU 或每个 tensor-parallel shard 启动一个 replica。HPA 不看 CPU，而是看 queue-wait。FP8，Marlin，和 INT4，AWQ，quantization 能把 GPU memory 控制在 H100 / H200 的容量范围内。端到端最终报告必须覆盖 throughput、acceptance rate、在 batch 1 / 8 / 32 下的 p50 / p99，以及 `$ / 1M tokens`。

## Architecture

```
request ingress
    |
    v
vLLM server (0.7) or SGLang (0.4)
    |
    +-- draft: EAGLE-3 heads | P-EAGLE parallel | ngram fallback
    +-- target: Llama 3.3 70B | Qwen3-Coder-30B | GPT-OSS-120B
    |     quantized FP8-Marlin or INT4-AWQ
    |
    v
verify pass: batch k draft tokens through target
    |
    v (accept prefix; resample for rejected suffix)
    v
token stream back to client
    |
    v
Prometheus metrics: throughput, acceptance rate, queue wait, latency p50/p99
    |
    v
HPA on queue-wait metric
```

## Stack

- Serving: vLLM 0.7 或 SGLang 0.4
- Speculative methods: EAGLE-3 draft heads、P-EAGLE parallel speculation、ngram fallback
- Draft training: SpecForge，SGLang，或 Red Hat Speculators
- Target models: Llama 3.3 70B、Qwen3-Coder-30B MoE、GPT-OSS-120B
- Quantization: FP8，Marlin，INT4 AWQ
- Deployment: Kubernetes + NVIDIA device plugin，基于 queue-wait metric 的 HPA
- Eval: ShareGPT、MT-Bench-v2、GSM8K、HumanEval，用于测 acceptance rate 在不同领域流量上的表现
- Reference: TensorRT-LLM speculative decoding，作为厂商基线

## Build It

1. **Target model prep.** 选择 Llama 3.3 70B。用 Marlin 量化到 FP8。通过 vLLM 0.7 部署在 1xH100 上，或 2x tensor-parallel。

2. **Draft source.** 从 Red Hat Speculators 拉取一个对齐好的 EAGLE-3 draft head，或用 SpecForge 自己训练一个。将其加载进 vLLM 的 speculative-decoding config。

3. **Baseline numbers.** 在启用 speculation 之前，测量 batch 1 / 8 / 32 下的 tokens/s、p50 / p99 latency 和 GPU utilization。公开这些基线数据。

4. **Enable EAGLE-3.** 打开配置，用同样的 benchmark 重跑。报告 speedup、acceptance rate，以及 p99 tail-latency 的变化。

5. **P-EAGLE.** 启用并行推测。比较更深 draft tree 的 P-EAGLE 与串行 EAGLE-3。报告它从何处开始带来收益，又从何处开始带来伤害。

6. **Domain traffic.** 用 ShareGPT、HumanEval 和领域专属流量跑同一台 server。测量每种分布下的 acceptance rate。找出 drafts 开始漂移的条件。

7. **Second target model.** 在 Qwen3-Coder-30B MoE 上运行同样的流水线。由于 MoE routing noise，draft 更难处理。把结果报告出来。

8. **K8s HPA.** 把它部署到 K8s，并让 HPA 追踪 `queue_wait_ms`。演示当负载变成三倍时系统如何横向扩容。

9. **Cost comparison.** 在相同 eval 上，计算与 Anthropic Claude Sonnet 4.7 和 OpenAI GPT-5.4 相比的 `$ / 1M tokens`。把结果发布出来。

## Use It

```
$ curl https://infer.example.com/v1/chat/completions -d '{"messages":[...]}'
[serve]     vLLM 0.7, Llama 3.3 70B FP8, EAGLE-3 active
[decode]    bs=8, accepted_tokens_per_step=3.2, acceptance_rate=0.76
[latency]   first-token 42ms, full-response 980ms (620 tokens)
[cost]      $0.34 per 1M output tokens at sustained throughput
```

## Ship It

`outputs/skill-inference-server.md` 描述了可交付成果。它是一套经过测量的 serving stack，包含 speculative decoding、完整 benchmark 报告和 K8s 部署。

| Weight | Criterion | How it is measured |
|:-:|---|---|
| 25 | Measured speedup vs baseline | 在两个模型上，以匹配质量实现 2.5x 以上吞吐提升 |
| 20 | Acceptance rate on realistic traffic | 按流量分布拆分的 acceptance-rate 报告 |
| 20 | P99 tail-latency discipline | 在 batch 1 / 8 / 32 下，对比启用与不启用 speculation 的 p99 |
| 20 | Ops | K8s 部署、基于 queue-wait 的 HPA、平滑 rollout |
| 15 | Write-up and methodology | 对变化内容与原因的清晰解释 |
| **100** | | |

## Exercises

1. 测量当 draft 落后 target 一个版本时 acceptance-rate 的退化情况，比如 Llama 3.3 到 3.4 的漂移。为此构建一个监控告警。

2. 实现 ngram-fallback。如果 EAGLE-3 的 acceptance 低于阈值，就切换到 ngram drafts。报告可靠性的提升。

3. 做一个可控 MoE 实验。在同一个 Qwen3-Coder-30B 上，对比注入 routing noise 与不注入 routing noise 的情况。测量 draft acceptance 对此的敏感度。

4. 扩展到 H200，141 GB。报告每个 replica 能多容纳多少模型尺寸，以及是否能在不量化的情况下服务 Llama 3.3 70B。

5. 在同样的 H100 硬件上 benchmark TensorRT-LLM 的 speculative decoding。报告它在哪些场景优于 vLLM。

## Key Terms

| Term | What people say | What it actually means |
|------|-----------------|------------------------|
| Draft model | “Speculator” | 为 target 提前提出 N 个 tokens 的小模型 |
| EAGLE-3 | “2026 draft architecture” | 在 target hidden states 上训练的 draft head，acceptance 大约 75% |
| P-EAGLE | “Parallel speculation” | 在一次 target pass 中验证多条 draft 分支组成的树 |
| Acceptance rate | “Hit rate” | 无需重采样而被接受的 drafted tokens 占比 |
| Quantization | “FP8 / INT4” | 通过低精度权重把更多模型塞进 GPU memory |
| Queue wait | “HPA metric” | 请求在实际开始推理前，在待处理队列中的等待时间 |
| Speculators hub | “Aligned drafts” | Red Hat Neural Magic 为常见开源模型提供的 EAGLE draft hub |

## Further Reading

- [vLLM EAGLE and P-EAGLE documentation](https://docs.vllm.ai) — 参考 serving stack
- [P-EAGLE (AWS 2026)](https://aws.amazon.com/blogs/machine-learning/p-eagle-faster-llm-inference-with-parallel-speculative-decoding-in-vllm/) — 并行 speculative decoding 论文与集成说明
- [SGLang SpecForge](https://github.com/sgl-project/SpecForge) — draft-head 训练流水线
- [Red Hat Speculators](https://github.com/neuralmagic/speculators) — 对齐 drafts 中心
- [TensorRT-LLM speculative decoding](https://nvidia.github.io/TensorRT-LLM/) — 厂商替代方案
- [Fireworks.ai serving architecture](https://fireworks.ai/blog) — 商业参考架构
- [EAGLE-3 paper (arXiv:2503.01840)](https://arxiv.org/abs/2503.01840) — 方法论文
- [vLLM repository](https://github.com/vllm-project/vllm) — 代码与 benchmarks
