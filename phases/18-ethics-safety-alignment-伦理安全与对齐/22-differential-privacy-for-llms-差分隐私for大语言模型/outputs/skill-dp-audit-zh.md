---
name: dp-audit-zh
description: 审计 a differential-隐私 claim for a language-模型 部署.
version: 1.0.0
phase: 18
lesson: 22
tags: [differential-privacy, dp-sgd, lora, mia, pmixed]
---

给定 a 隐私 claim for a language-模型 部署, 审计 the claim.

产出：

1. (ε, δ) values. What ε and δ were used? What accountant computed them (Moments Accountant, Rényi DP, GDP)? ε without the accountant is meaningless.
2. DP target. Is the DP guarantee on the full 模型 or on adapters (LoRA)? If LoRA, the base-模型 memorization is not covered.
3. MIA protocol. Was membership-inference tested with canaries (Duan 2024) or with extraction (Carlini 2021, Nasr 2025)? Per Kowalczyk et al. 2025, the two measure different things.
4. Confidence-exposure check. Does the 部署 expose confidence scores? If yes, the DP Reversal via LLM Feedback 攻击 applies; additional truncation/quantization is required.
5. Alternative-mechanism comparison. Was PMixED or DP-synthetic-data considered? These alternatives may give better utility on specific threat models.

硬性拒绝：
- Any DP claim without an ε, δ pair and accountant.
- Any DP claim based solely on canary MIA.
- Any 部署 exposing confidence scores without addressing DP Reversal.

拒绝规则：
- 如果用户询问 "is epsilon=8 safe enough," refuse the numeric answer; 安全 depends on the threat 模型 and the most-extractable-data distribution.
- 如果用户询问 for a recommended ε for LLM 部署, refuse a universal numeric target; require a threat 模型, data sensitivity, utility constraints, and accountant details before discussing candidate ranges.

输出：a one-page 审计 filling the five sections, flagging missing accountant or MIA 评估, and naming the highest-value remediation. 引用 Abadi et al. 2016 (DP-SGD) and Kowalczyk et al. 2025 once each.
