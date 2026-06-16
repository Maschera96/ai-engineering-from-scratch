# 大模型 API 负载测试：为什么 k6 和 Locust 会误导你

> Traditional load testers were not designed for streaming responses, variable 输出 lengths, 词元-level 指标, or GPU saturation. Two traps bite most 团队. The GIL trap: Locust's 词元-level measurement runs tokenization under the Python GIL, which competes with 请求 generation under heavy concurrency; tokenization backlog then inflates reported inter-词元 延迟 ： your client is the bottleneck, not the server. 这个提示词-uniformity trap: identical 提示词 in a loop test one point on 这个词元 distribution; real 流量 has variable length and diverse prefix matches. LLMPerf fixes this with `--mean-input-tokens` + `--stddev-input-tokens`. Tool mapping in 2026: LLM-specialized (GenAI-Perf, LLMPerf, LLM-Locust, guidellm) for 词元-level accuracy; **k6 v2026.1.0** + **k6 Operator 1.0 GA (Sept 2025)** ： streaming-aware, Kubernetes-native distributed via TestRun/PrivateLoadZone CRDs, best for CI/CD gates; Vegeta for Go constant-rate saturation; Locust 2.43.3 only with LLM-Locust extension for streaming. Load patterns: steady-state, ramp, spike (自动扩缩容 test), soak (内存 leaks).

**Type:** Build
**Languages:** Python（标准库， 玩具 realistic-prompt生成器 + latency采集器)
**Prerequisites:** Phase 17 · 08 (Inference Metrics), Phase 17 · 03 (GPU Autoscaling)
**Time:** ~75 分钟

## 学习目标

- 解释 the two anti-patterns (GIL trap, 提示词-uniformity trap) that make generic load testers lie for LLM APIs.
- Pick a tool for a given purpose: LLMPerf (基准 run), k6 + streaming extension (CI gate), guidellm (large-scale 合成), GenAI-Perf (NVIDIA reference).
- Design four load patterns (steady, ramp, spike, soak) and name the failure mode each catches.
- Build a realistic 提示词 distribution using 均值 + stddev of 输入 词元 rather than fixed length.

## 问题

You k6-tested your LLM endpoint at 500 concurrent users. It held. You shipped. In 生产化 at 200 actual users the service fell over ： P99 TTFT exploded, GPUs pinned.

Two things happened. First, k6 sent 500 identical 提示词 ： your 请求-coalescing and prefix caching made it look like you were handling 500 concurrent decodes when you were actually handling one. Second, k6 doesn't track inter-词元 延迟 on streaming responses the way the eye experiences it; it sees one HTTP connection, not 500 词元 arriving at varying intervals.

负载测试 for LLMs is its own discipline.

## 概念

### GIL 陷阱（Locust）

Locust uses Python and runs tokenization client-side under the GIL. Under high concurrency the tokenizer queues behind 请求 generation. Reported inter-词元 延迟 includes client-side tokenization backlog. You think the server is slow; it's the test harness.

Fix: LLM-Locust extension moves tokenization to separate processes, or use a compiled-language harness (k6, LLMPerf using tokenizers.rs).

### 这个提示词-uniformity trap

All known load testers let you configure one 提示词. In a loop test of 10,000 iterations the exact same 提示词 sends each time. Server sees the same prefix every time ： 前缀缓存 hits approach 100%, 吞吐量 looks great.

Fix: sample from 一个提示词 distribution. LLMPerf uses `--mean-input-tokens 500 --stddev-input-tokens 150` ： diverse lengths, diverse content.

### 四种负载模式

1. **Steady-state** ： constant RPS for 30-60 min. Catches: baseline performance regressions.
2. **Ramp** ： linearly increase RPS from 0 to target over 15 min. Catches: 容量 breakpoint, warm-up anomalies.
3. **Spike** ： sudden 3-10x RPS for 2 min then back. Catches: 自动扩缩容 延迟, 队列 saturation, cold-start impact.
4. **Soak** ： steady-state for 4-8 hours. Catches: 内存 leaks, connection-pool drift, 可观测性 overflow.

### 2026 tool mapping

**LLMPerf** (Anyscale) ： Python but Rust-backed tokenization. 均值/stddev 提示词. Streaming-aware. Best 默认 for performance runs.

**NVIDIA GenAI-Perf** ： NVIDIA's reference. Uses Triton client; comprehensive 指标 coverage. Note its ITL excludes TTFT; LLMPerf's includes it. Two tools produce different TPOT for the same server.

**LLM-Locust** (TrueFoundry) ： Locust extension that fixes the GIL trap. Familiar Locust DSL + streaming 指标.

**guidellm** ： large-scale 合成 benchmarking.

**k6 v2026.1.0** + **k6 Operator 1.0 GA (Sept 2025)**:
- k6 itself (Go, compiled, no GIL) 新增 streaming-aware 指标.
- k6 Operator uses TestRun / PrivateLoadZone CRDs for Kubernetes-native distributed testing.
- Best for CI/CD gates and SLA testing.

**Vegeta** ： Go, simpler than k6. Constant-rate HTTP saturation. Not LLM-aware but good for 网关 / 限流 testing.

**Locust 2.43.3 stock** ： has the GIL trap for LLM. Only with LLM-Locust extension.

### CI 中的 SLA 关卡

运行k6 on the PR with:

- 30-50 iterations each at baseline RPS.
- Gate: P50/P95 TTFT, 5xx < 5%, TPOT under threshold.
- Break the build on breach.

### 真实提示词分布

Build from real 流量 samples (if you have them) or from published distributions (e.g., ShareGPT 提示词 for chat, HumanEval for code). Feed 这个均值 + stddev to LLMPerf. Avoid loop-with-one-提示词 at all costs.

### 你应该记住的数字

- k6 Operator 1.0 GA: September 2025.
- k6 v2026.1.0: streaming-aware 指标.
- Typical LLMPerf run: 100-1000 请求 at concurrency X.
- Typical CI gate: 30-50 iterations per PR.
- Four patterns: steady, ramp, spike, soak.

## 使用它

`code/main.py` simulates a load test with realistic 提示词 distribution, measures effective TPOT, and demonstrates the uniform-提示词 trap.

## 交付它

This lesson 产出 `outputs/skill-load-test-plan.md`. 给定工作负载 and SLA, picks tool and designs the four load patterns.

## 练习

1. Run `code/main.py`. 比较 uniform 对比 realistic distribution ： where is the gap?
2. Write the k6 script for a CI gate: TTFT P95 < 800 ms at 100 concurrent, runtime 5 minutes.
3. Your soak test shows 内存 growing 50 MB/小时. Name three causes and 这个埋点 to pick between them.
4. Spike test from 10 RPS to 100 RPS. What's the expected recovery time if Karpenter + vLLM 生产化-stack are in place (Phase 17 · 03 + 18)?
5. GenAI-Perf reports TPOT=6ms; LLMPerf reports TPOT=11ms on the same server. 解释.

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| LLMPerf | "the LLM harness" | Anyscale 基准 tool, streaming-aware |
| GenAI-Perf | "NVIDIA tool" | NVIDIA reference harness |
| LLM-Locust | "Locust for LLMs" | Locust extension fixing GIL trap |
| guidellm | "合成 基准" | Large-scale 合成 tool |
| k6 Operator | "K8s k6" | CRD-based distributed k6 |
| GIL trap | "Python client overhead" | Tokenization backlog inflates reported 延迟 |
| 提示词-uniformity trap | "single-提示词 lie" | Loop with same 提示词 hits cache, inflates 吞吐量 |
| Steady-state | "constant load" | Flat RPS for N minutes |
| Ramp | "linear up" | 0 to target over duration |
| Spike | "burst test" | Sudden multiplier then revert |
| Soak | "long test" | Hours for leak detection |

## 延伸阅读

- [TianPan — Load Testing LLM Applications](https://tianpan.co/blog/2026-03-19-load-testing-llm-applications)
- [PremAI — Load Testing LLMs 2026](https://blog.premai.io/load-testing-llms-tools-metrics-realistic-traffic-simulation-2026/)
- [NVIDIA NIM — Introduction to LLM Inference Benchmarking](https://docs.nvidia.com/nim/large-language-models/1.0.0/benchmarking.html)
- [TrueFoundry — LLM-Locust](https://www.truefoundry.com/blog/llm-locust-a-tool-for-benchmarking-llm-performance)
- [LLMPerf](https://github.com/ray-project/llmperf)
- [k6 Operator](https://github.com/grafana/k6-operator)
