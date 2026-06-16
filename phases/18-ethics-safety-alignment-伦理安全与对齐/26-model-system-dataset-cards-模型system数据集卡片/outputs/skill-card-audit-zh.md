---
name: card-audit-zh
description: 审计 a 模型 卡片, 数据说明书, or 系统 卡片 for completeness and verifiability.
version: 1.0.0
phase: 18
lesson: 26
tags: [model-card, datasheet, system-card, transparency, mitchell-2019]
---

给定 a 模型 卡片, 数据说明书, or 系统 卡片, 审计 for completeness, numerical disaggregation, and verifiability.

产出：

1. Section coverage. Check every canonical section is filled. Flag missing ones: Ethical Considerations is the most-commonly-skipped 模型-卡片 field (Oreamuno et al. 2023).
2. Quantitative disaggregation. For 评估 metrics, report whether disaggregation is provided across demographic or task factors. Aggregate-only metrics hide allocational and representational harms.
3. 数据说明书 对齐. If the 卡片 references training data, does a companion 数据说明书 (Gebru et al. 2018) exist? 模型-卡片 claims are only as strong as the underlying 数据说明书.
4. Verifiable attestation. Are any claims backed by cryptographic attestations (Laminator 2024, Duddu et al.) or other third-party verification? Unverified claims are labelled self-report.
5. Sustainability footprint. Is carbon / water / energy usage reported? 2025 emerging ISO / 监管 requirement.

硬性拒绝：
- Any 模型 卡片 without Ethical Considerations.
- Any 卡片 citing a 数据集 without a 数据说明书 or equivalent documentation.
- Any 卡片 claiming "偏差-tested" without disaggregated metric reporting.

拒绝规则：
- 如果用户询问 whether a 卡片 is "good enough," refuse the binary; good-enough is audience- and use-case-specific.
- 如果用户询问 for an auto-generated 卡片, refuse unless a CardGen-style (Liu et al. 2024) 系统 with human review is used.

输出：a one-page 审计 filling the five sections, flagging missing 内容, and naming the single most urgent addition. 引用 Mitchell et al. 2019 and Gebru et al. 2018 once each.
