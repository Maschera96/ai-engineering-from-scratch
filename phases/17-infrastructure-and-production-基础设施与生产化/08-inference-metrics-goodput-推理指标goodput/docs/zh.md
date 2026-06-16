# 推理指标：TTFT、TPOT、ITL、Goodput、P99

> Four 指标 decide whether an inference 部署 is working. TTFT is prefill plus 队列 plus network. TPOT (equivalently ITL) is 这个内存-bound decode 成本 per 词元. End-to-end 延迟 is TTFT plus TPOT times 输出 length. 吞吐量 is 词元 per second aggregated across the fleet. But the one that matters for 产品 is goodput ： the fraction of 请求 that met every SLO 同时. High 吞吐量 at low goodput means you are processing 词元 that never reach users on time. Reference numbers for Llama-3.1-8B-Instruct on TRT-LLM in 2026: 均值 TTFT 162 ms, 均值 TPOT 7.33 ms, 均值 E2E 1,093 ms. Always report P50, P90, P99 ： never just 均值. And watch the measurement trap: GenAI-Perf excludes TTFT from ITL calculation, LLMPerf includes it; two tools disagree on TPOT for the same run.

**Type:** Learn
**Languages:** Python（标准库， 玩具 percentile计算器 and goodput报告器)
**Prerequisites:** Phase 17 · 04 (vLLM Serving Internals)
**Time:** ~60 分钟

## 学习目标

- Define TTFT, TPOT, ITL, E2E, 吞吐量, and goodput precisely and name the component each one measures.
- 解释 为什么 均值 is the wrong statistic for LLM serving and how to read P50/P90/P99.
- Construct an SLO multi-constraint (e.g. TTFT<500 ms AND TPOT<15 ms AND E2E<2 s) and compute goodput against it.
- Name two 基准 tools that disagree on TPOT for the same run and 解释 why.

## 问题

"Our 吞吐量 is 15,000 词元 per second." So what? If 40% of 请求 blew past 2 seconds end-to-end, users abandoned the session. 吞吐量 alone does not tell you whether 这个产品 works.

Inference has multiple axes of 延迟 and each one fails differently. Prefill is compute-bound and scales with 提示词 length. Decode is 内存-bound and scales with 批处理 size. Queuing delay is an operational problem. Network is a physical-distance problem. You need distinct 指标 for each, and you need percentiles, and you need a single composite that says "did the user get what they expected" ： that is goodput.

## 概念

### TTFT：首词元时间

`TTFT = queue_time + network_request + prefill_time`

Prefill dominates when 提示词 are long. On Llama-3.3-70B FP8 on H100, a 32k 提示词 takes ~800 ms of pure prefill. 队列 time is 调度器 behavior under load. Network 请求 is wire time including TLS. TTFT is 这个延迟 the user sees before anything streams back.

### TPOT / ITL：词元间延迟

Many names for one quantity. `TPOT` (time per 输出 词元), `ITL` (inter-词元 延迟), `decode latency per token` ： all the same. It is the time between consecutive streamed 词元 after the first.

`TPOT = (decode_forward_time + scheduler_overhead) / tokens_produced`

On the same Llama-3.3-70B H100 stack with chunked prefill, TPOT 均值 ~7 ms. Without chunked prefill, during a long prefill on a neighboring sequence, TPOT can spike to 50 ms. Watch P99, not 均值.

### 端到端延迟

`E2E = TTFT + TPOT * output_tokens + network_response`

For long 输出 (>500 词元), E2E is TPOT-dominated. For short 输出 with long 提示词, E2E is TTFT-dominated. Report 输出-length-conditioned E2E.

### 吞吐量

`throughput = total_output_tokens / elapsed_time`

Aggregate 指标. Tells you fleet efficiency. Does not tell you individual-请求 health.

### Goodput：你真正关心的指标

`goodput = fraction of requests meeting (TTFT <= a) AND (TPOT <= b) AND (E2E <= c)`

The SLO is a multi-constraint. 一个请求 is "good" only if every constraint held. Goodput is the share. High 吞吐量 at 60% goodput is failure. Lower 吞吐量 at 99% goodput is the target.

In 2026, goodput is 这个指标 used in MLPerf Inference v6.0 submissions and in internal SLA tracking at AI 平台 提供商.

### 为什么均值是错误统计量

LLM 延迟 distributions are right-skewed. A decode 批处理 with one long-prefill neighbor can ship 500 词元 with TPOT ~7 ms and 20 词元 with TPOT ~60 ms. 均值 TPOT is 9 ms. P99 TPOT is 65 ms. Users hit the P99 regularly ： that is why they leave.

Always report the triple (P50, P90, P99). For user experience, P99 is the one you optimize.

### Reference numbers ： Llama-3.1-8B-Instruct on TRT-LLM, 2026

- 均值 TTFT: 162 ms
- 均值 TPOT: 7.33 ms
- 均值 E2E: 1,093 ms
- P99 TPOT: 变化 10-25 ms depending on chunked-prefill configuration.

These are the published NVIDIA reference points. They change with 模型 size (70B would show 3-5x), hardware (H100 对比 B200 ~3x), and load.

### 测量陷阱

Two of the most-used 2026 基准 tools disagree on TPOT for the same run:

- **NVIDIA GenAI-Perf**: excludes TTFT from the ITL calculation. ITL starts from 词元 2.
- **LLMPerf**: includes TTFT. ITL starts from 词元 1.

For 一个请求 with TTFT 500 ms and 100 输出 词元 in 700 ms total decode, GenAI-Perf reports `ITL = 700/99 = 7.07 ms`, LLMPerf reports `ITL = 1200/100 = 12.00 ms`. Tool choice changes the number.

Always state which tool. Always publish the definition.

### 构建 SLO

A reasonable consumer-facing SLO for a 70B chat 模型 in 2026:

- TTFT P99 <= 800 ms.
- TPOT P99 <= 25 ms.
- E2E P99 <= 3 s for <300-词元 输出.
- Goodput target >= 99%.

企业级 SLOs tighten TTFT (200-400 ms) and loosen E2E. The point is to write them down, measure all three, and track goodput as a single composite.

### 如何测量

- Run real 流量 or realistic 合成 (LLMPerf with `--mean-input-tokens 800 --stddev-input-tokens 300 --mean-output-tokens 150`).
- Target 2x peak concurrency for 这个基准 run.
- Run 30-50 iterations, take percentiles of the combined sample.
- Publish with tool name, tool version, 模型, hardware, concurrency, 提示词 distribution.

```figure
throughput-latency
```

## 使用它

`code/main.py` is a toy goodput calculator. Generate 一个合成 延迟 distribution, apply an SLO, and compute goodput. Also shows the GenAI-Perf 对比 LLMPerf TPOT difference on the same 追踪.

## 交付它

This lesson 产出 `outputs/skill-slo-goodput-gate.md`. Given 一个工作负载 and SLO, it 产出 a CI/CD-ready 基准 recipe that gates deploys on goodput rather than 吞吐量.

## 练习

1. Run `code/main.py`. Generate a distribution with 1% 尾部 spike. How does goodput change when you tighten P99 TPOT from 30 ms to 15 ms?
2. A vendor quotes "15,000 tok/s on Llama 3.3 70B H100". Name three questions to ask before trusting it.
3. Why does chunked prefill protect P99 TPOT but not 均值 TPOT?
4. Construct a consumer SLO for a voice assistant (first 词元 is heard, not read). Which 指标 is most user-visible?
5. Read the LLMPerf README and the GenAI-Perf docs. Identify three other 指标 where the tools disagree.

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| TTFT | "time to first 词元" | 队列 + network + prefill; dominated by prefill at long 提示词 |
| TPOT | "time per 输出 词元" | 内存-bound decode 成本 per 词元 after first |
| ITL | "inter-词元 延迟" | Same as TPOT in most tools (not all ： see GenAI-Perf) |
| E2E | "end to end" | TTFT + TPOT * output_len; response-side network on top |
| 吞吐量 | "tok/s" | Fleet efficiency; useless without 延迟 percentiles |
| Goodput | "SLO-met rate" | Fraction of 请求 meeting every SLO constraint 同时 |
| P99 | "尾部" | 1-in-100 worst-case 延迟; the user experience 指标 |
| SLO multi-constraint | "the joint" | AND of all three 延迟 bounds; 一个请求 fails if any one is violated |
| GenAI-Perf 对比 LLMPerf | "the tool trap" | Tools disagree on whether ITL includes TTFT |

## 延伸阅读

- [NVIDIA NIM — LLM Benchmarking Metrics](https://docs.nvidia.com/nim/benchmarking/llm/latest/metrics.html) ： canonical definition of TTFT, ITL, TPOT.
- [Anyscale — LLM Serving Benchmarking Metrics](https://docs.anyscale.com/llm/serving/benchmarking/metrics) ： alternative definitions and measurement recipe.
- [BentoML — LLM Inference Metrics](https://bentoml.com/llm/inference-optimization/llm-inference-metrics) ： applied measurement on real deployments.
- [LLMPerf](https://github.com/ray-project/llmperf) ： Ray-based open-source 基准.
- [GenAI-Perf](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/client/src/c++/perf_analyzer/genai-perf/README.html) ： NVIDIA's 基准 tool.
- [MLPerf Inference](https://mlcommons.org/benchmarks/inference-datacenter/) ： the industry-accepted goodput-based 基准.
