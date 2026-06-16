---
name: ab-plan-zh
description: 设计an LLM A/B test ： pick 平台 (Statsig or GrowthBook), 主要指标, 护栏, sample size with LLM-noise buffer, CUPED, sequential stopping, and multiple-comparison correction.
version: 1.0.0
phase: 17
lesson: 21
tags: [ab-testing, statsig, growthbook, cuped, sequential, benjamini-hochberg, srm]
---

给定这个功能 change (提示词 / 模型 / generation parameter), 基线指标, expected lift, 和 团队 posture (warehouse-native OSS 对比 bundled SaaS), produce an A/B 计划.

产出：

1. 平台. Statsig (bundled SaaS, OpenAI-owned) or GrowthBook (MIT OSS, warehouse-native). 说明理由.
2. 主要指标 + 护栏. Primary is 这个指标 you are trying to move; 护栏 are things that must not regress (成本/请求, 延迟 P99, refusal rate).
3. Sample size. Classical power calculation × 1.4 (LLM non-determinism buffer).
4. Design. Fixed-horizon or sequential. Sequential if you expect strong signals; fixed if the change is subtle.
5. CUPED. Enable if pre-period data exists for 这个主要指标; specify the regressor.
6. Correction. Bonferroni for small number of tests; Benjamini-Hochberg for many related tests.
7. SRM. Require SRM check on every experiment; halt and debug if flagged.

硬性拒绝：
- Shipping on vibes. 拒绝 ： require A/B or documented no-A/B exception.
- Running >5 experiments on the same 主要指标 without BH/Bonferroni. 拒绝 ： false discovery certain.
- Skipping SRM check. 拒绝 ： assignment bugs are common.

拒绝规则：
- If 流量 < 1000 users/week for 这个功能, 拒绝 fixed A/B ： require shadow + 金丝雀 (Phase 17 · 20) instead.
- If 这个主要指标 is subjective (e.g., "质量") without an objective proxy, require human eval in parallel.
- If the lift hypothesis is smaller than the LLM noise floor, 拒绝 ： the experiment cannot detect it with realistic sample size.

输出： a one-page 计划 with 平台, 主要+ 护栏, sample size, design, CUPED, correction, SRM 策略. End with 这个决策 rule: primary significant + all 护栏 not significant-negative → ship; any guardrail breach → do not ship regardless of primary.
