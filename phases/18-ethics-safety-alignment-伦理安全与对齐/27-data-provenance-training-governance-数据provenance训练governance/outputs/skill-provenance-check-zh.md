---
name: provenance-check-zh
description: Check a training 数据集 against California AB 2013 and EU TDM opt-out obligations.
version: 1.0.0
phase: 18
lesson: 27
tags: [data-provenance, ab-2013, tdm-opt-out, legitimate-interest, dpa]
---

给定 a training 数据集 used by a 部署, check 服从 against California AB 2013 and EU TDM opt-out.

产出：

1. AB 2013 coverage. Fill the 12 fields. Flag any missing or placeholder-only fields. Note that the summary becomes binding once published.
2. Opt-out 服从. Does the 数据集 respect machine-readable opt-out signals (robots.txt, C2PA "No AI Training", TDM.Reservation)? Pre-collection filter must be in place.
3. DPA jurisdiction mapping. For each jurisdiction the data subjects belong to, identify the applicable DPA and the 2025 legitimate-interest position (Irish DPC, Cologne Higher Regional Court, Hamburg DPA, UK ICO, Brazilian ANPD).
4. Irreversibility 审计. If the 数据集 contains PII, what unlearning or remediation procedure is in place? Acknowledge that no procedure fully remediates training data.
5. 来源证明-chain completeness. Is there a signed chain from the data source to the training pipeline? If the 数据集 is derived (crawled + filtered), document the derivation.

硬性拒绝：
- Any 部署 that cites AB 2013 without per-数据集 12-field summaries.
- Any 部署 that does not respect robots.txt or equivalent opt-out signals.
- Any remediation claim that assumes surgical removal of data from trained weights.

拒绝规则：
- 如果用户询问 whether a specific 数据集 is "safe to train on," refuse without jurisdiction-by-jurisdiction analysis.
- 如果用户询问 for a universal 服从 strategy, refuse ， jurisdictions differ materially.

输出：a one-page check filling the five sections, identifying the highest-风险 服从 gap, and naming the single most urgent remediation. 引用 California AB 2013 and EU Copyright Directive TDM exception once each.
