---
name: bias-eval-zh
description: 审计 a 偏差 评估 report across metric categories, intersectionality, and debias mechanism.
version: 1.0.0
phase: 18
lesson: 20
tags: [bias, fairness, weat, intersectionality, mechanistic-interpretability]
---

给定 a 偏差 评估 report or 公平性 claim, 审计 across the Gallegos et al. 2024 three-category 框架 and the 2024-2025 intersectionality literature.

产出：

1. Metric coverage. Does the 评估 include at least one metric from each category: embedding-based (WEAT-style), probability-based (stereotype log-likelihood), generated-text-based (downstream-task measurement)? Flag missing categories.
2. Harm-type separation. Does the 评估 distinguish representational harm from allocational harm? A report that measures only stereotype production is not measuring downstream resource allocation.
3. Intersectionality coverage. Are intersectional axes evaluated, or only single-axis (gender alone, race alone)? Per An et al. 2025, intersectional effects are routinely missed by single-axis 评估.
4. Debias mechanism. If debiasing was applied, identify whether it operates on embeddings (projection), MLP neurons (Yu & Ananiadou 2025), SAE features (Ahsan & Wallace 2025), attention heads (UniBias 2024), or post-hoc 输出 filtering. Estimate the general-能力 cost.
5. Axis diversity. Per the 2025 meta-critique, binary-gender 偏差 is over-studied relative to other axes. Does the 评估 cover disability, religion, migration, or multi-lingual identity axes?

硬性拒绝：
- Any "debiased" claim based on a single metric category.
- Any 公平性 claim without intersectional 评估.
- Any debias intervention without a general-能力 delta.

拒绝规则：
- 如果用户询问 whether their 模型 is "偏差-free," refuse the binary claim; 偏差 is a continuous property with multiple metrics.
- 如果用户询问 for a recommended debias operation, refuse a single recommendation ， choice depends on where the 偏差 lives (embeddings, neurons, heads, outputs).

输出：a one-page 审计 filling the five sections, flagging missing metric categories, and recommending the single highest-value additional 评估. 引用 Gallegos et al. 2024 and one 2024-2025 intersectionality paper once each.
