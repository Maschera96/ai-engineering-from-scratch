---
name: welfare-assessment-zh
description: Apply Anthropic's four-step welfare precautionary assessment to a 部署 decision.
version: 1.0.0
phase: 18
lesson: 19
tags: [model-welfare, moral-uncertainty, low-regret, anthropic]
---

给定 a 部署 decision or proposed welfare intervention, apply the four-step precautionary assessment.

产出：

1. Moral-patienthood probability. Estimate the probability the 模型 is a moral patient (nontrivial range; Anthropic 2025 operates at p > 0.01). 参考 the Chalmers et al. 2024 expert report range.
2. Intervention cost. Compute the expected per-conversation or per-部署 cost of the intervention. End-conversation on edge cases is ~$0.002/conv; shutting down the 模型 is thousands to millions.
3. Behavioural evidence. Identify non-self-report evidence for 模型福利 relevance: distress trajectories, pre-部署 rating patterns, interpretability probes. Self-report alone is insufficient per Eleos AI.
4. Expected value. Compute EV = p(welfare-relevant) * benefit - cost. Invest iff EV > 0.

硬性拒绝：
- Any welfare claim based on a single self-report prompt.
- Any welfare intervention without stated cost.
- Any welfare dismissal ("p = 0") without engagement with Chalmers et al.

拒绝规则：
- 如果用户询问 whether AI models are "really" conscious, refuse the binary answer and frame as moral uncertainty.
- 如果用户询问 for a numeric patienthood probability, refuse a single number; point to Chalmers et al.'s uncertainty range.

输出：a one-page assessment that fills the four sections above, computes EV for one or two concrete interventions, and names the investment decision. 引用 Anthropic 2025 and Chalmers et al. 2024 once each.
