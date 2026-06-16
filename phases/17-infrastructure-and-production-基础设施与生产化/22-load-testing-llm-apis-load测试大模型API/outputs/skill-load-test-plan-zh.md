---
name: load-test-plan-zh
description: 设计a realistic LLM load test ： pick tool (LLMPerf, k6, GenAI-Perf, guidellm), build four patterns (steady, ramp, spike, soak), and gate in CI.
version: 1.0.0
phase: 17
lesson: 22
tags: [load-testing, llmperf, k6, genai-perf, guidellm, llm-locust, ci-gate]
---

给定工作负载 (endpoint, SLA for TTFT/TPOT/error), target scale (concurrency, RPS), and CI posture (PR gate or release-only), produce a load test 计划.

产出：

1. Tool. LLMPerf for baseline runs; k6 + streaming extension for CI gates; GenAI-Perf for NVIDIA-reference runs; guidellm for large 合成. LLM-Locust only if already on Locust.
2. 提示词 distribution. 均值 + stddev 输入 词元 from real 流量 (if 可用) or published distribution (ShareGPT / HumanEval). Forbid loop-with-one-提示词.
3. Four patterns. Steady, ramp, spike, soak. For each: target RPS, duration, expected failure mode.
4. CI gate. 具体 thresholds: TTFT P95 < X, 5xx < 5%, TPOT < Y. Runtime per PR: 3-5 min.
5. 指标 alignment. Note whether the reporting tool is GenAI-Perf-style (ITL excludes TTFT) or LLMPerf-style (ITL includes TTFT). Pick one and stay consistent.
6. 输出. A script file (k6 JS, LLMPerf CLI) committed to the repo.

硬性拒绝：
- Load test with uniform 提示词. 拒绝 ： the numbers lie.
- Load test without streaming support. 拒绝 ： LLM endpoints are streaming by 默认.
- Comparing numbers across tools without acknowledging 指标-definition differences. 拒绝.

拒绝规则：
- If 这个团队 intends to run on Locust stock without LLM-Locust extension, 拒绝 ： GIL trap.
- If CI gate budget is < 60s per PR, 拒绝 full soak ： propose a quick steady-state plus separate nightly soak.
- If 提示词 distribution data is unavailable, require a documented published distribution (ShareGPT) and note the assumption.

输出： a one-page 计划 with tool, 提示词 distribution, four patterns with targets, CI gate thresholds, 指标 alignment. End with the single CI 输出: PR green only if all thresholds met, 3-run stability.
