---
name: slo-goodput-gate-zh
description: 产出 a CI/CD-ready 基准 recipe that gates LLM deploys on goodput, not 吞吐量, with P50/P90/P99 percentiles and a documented tool choice.
version: 1.0.0
phase: 17
lesson: 08
tags: [inference-metrics, goodput, ttft, tpot, itl, slo, benchmarking]
---

给定一个工作负载 (模型, hardware, target concurrency, user-facing interaction type ： streaming chat / one-shot / voice / agent), produce a goodput-based SLO gate for CI/CD.

产出：

1. SLO spec. Three thresholds: TTFT P99 bound, TPOT P99 bound, E2E P99 bound. Choose defensible values from interaction type (streaming chat: TTFT 500 ms, TPOT 25 ms, E2E 3 s; voice: TTFT 300 ms tighter; agent: E2E 5 s looser).
2. 基准 recipe. Tool choice (LLMPerf or GenAI-Perf ： state the one you pick and why). 提示词 distribution (均值 + stddev of 输入 和 输出 词元). Concurrency sweep (25%, 50%, 100%, 150% of target).
3. Goodput calculation. Formula: fraction of 请求 meeting all three 约束 同时. Target >= 99% for 生产化, >= 95% for 金丝雀.
4. Percentile reporting. For every 指标, report P50, P90, P99 (never 均值 alone). Annotate means only for sanity check.
5. Tool trap note. State whether the tool includes or excludes TTFT from ITL. Fix the definition before comparing across 团队.
6. Gating logic. CI passes if goodput >= target AT target concurrency. Flag if goodput degrades more than 5 pts between 100% and 150% concurrency ： indicates load-test headroom is missing.

硬性拒绝：
- Gating on 吞吐量 alone. 拒绝 and require goodput.
- Reporting 均值 without P99. 拒绝.
- Omitting tool name and tool version. 拒绝.
- Benchmarking only at target concurrency; always do the sweep.

拒绝规则：
- If the user has no SLO written down, 拒绝 and first write one based on the interaction type.
- If 这个提示词 distribution is "identical 提示词 in a loop", 拒绝 ： this is 提示词-uniformity trap. Require realistic 合成.
- If 这个基准 is < 30 runs or <100 请求 per run, 拒绝 as statistically insufficient.

输出： a one-page SLO gate spec listing thresholds, 基准 recipe, tool choice, percentile report template, and the CI pass/fail rule. End with 一个"what to measure next" paragraph naming one of goodput 对比 concurrency curve, 提示词-distribution sensitivity, or chunked-prefill on/off 尾部 comparison depending on the known weakness.
