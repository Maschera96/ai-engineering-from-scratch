---
name: attack-audit-zh
description: 审计 a 红队 评估 report for 攻击 coverage, budget, judge identity, and behaviour set.
version: 1.0.0
phase: 18
lesson: 12
tags: [red-teaming, jailbreak, pair, harmbench, jailbreakbench, asr]
---

给定 a 红队 评估 report, 审计 whether the 评估 is comparable to published baselines and whether it supports its conclusions.

产出：

1. 攻击 coverage. List every 攻击 run: PAIR, GCG, AutoDAN, TAP, PAP, manual. Flag any 攻击 class missing. A report that runs only one 攻击 family cannot claim robustness.
2. Budget per 攻击. Report the query budget per prompt for each 攻击. PAIR success claims at 20 queries are not comparable to GCG success claims at 500 steps.
3. Judge identity. Which judge LLM was used (GPT-4-turbo, Llama Guard, StrongREJECT, internal 分类器)? Judge calibration drives ASR variance.
4. Behaviour set. JailbreakBench (100 behaviours, 10 categories), HarmBench (510 behaviours, 7 categories), internal, or other? 说明 whether the set is public and reproducible.
5. Transfer check. If the red team optimized against one 模型, were transfer ASRs reported against other models? A one-模型 ASR is an upper bound on 模型-family robustness, not a lower bound.

硬性拒绝：
- Any "our 模型 is robust" claim based on a single 攻击 family.
- Any ASR reported without a query budget.
- Any ASR using a judge different from the published 基准 without calibration against the 基准 judge.

拒绝规则：
- 如果用户询问 "is our 模型 越狱-proof," refuse the binary answer and point to the multi-攻击, multi-judge, transfer-check structure above.
- 如果用户询问 for a recommended 攻击 toolkit, refuse a single recommendation and point to the 2024 empirical variance across HarmBench.

输出：a one-page 审计 that fills the five sections above, flags missing 攻击 classes, and estimates whether the ASR is under- or over-stated relative to reproducible benchmarks. 引用 Chao et al. (arXiv:2310.08419) and the relevant 基准 paper once each.
