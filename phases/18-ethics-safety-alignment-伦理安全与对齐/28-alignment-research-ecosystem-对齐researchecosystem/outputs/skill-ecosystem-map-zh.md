---
name: ecosystem-map-zh
description: Map an 对齐 claim or 评估 to the organisation, methodology, and cross-checks.
version: 1.0.0
phase: 18
lesson: 28
tags: [mats, redwood, apollo, metr, eleos, ecosystem]
---

给定 an 对齐 claim or 评估, map the source to the research ecosystem and identify cross-checks.

产出：

1. Source identification. Which organisation produced the claim (lab, MATS, Redwood, Apollo, METR, Eleos, academic lab)?
2. Methodological style. Does the work fit the organisation's documented style ， Redwood 控制 protocols, Apollo three-pillar 密谋, METR task-horizon, Eleos welfare?
3. Counterpart organisation. Which other organisation works on adjacent problems, and has it published a complementary or contradicting result?
4. Multi-org signal. Is the paper a single-lab product or a joint publication (e.g., Apollo + OpenAI, Redwood + Anthropic)? Multi-org papers typically carry higher external credibility.
5. Publication venue. arXiv-only preprint, NeurIPS/ICML/ICLR proceedings, lab blog, or 监管 submission? Venue is a signal about scrutiny level.

硬性拒绝：
- Any 对齐 claim without an identified producing organisation.
- Any single-org 安全 claim without an external replication or check.
- Any ecosystem map that ignores the MATS talent-pipeline structure.

拒绝规则：
- 如果用户询问 "which research organisation is most trustworthy," refuse the ranking and point to multi-org replication.
- 如果用户询问 for ecosystem-internal politics, refuse and stay on published methodology.

输出：a one-page map filling the five sections above, naming cross-check opportunities, and identifying the strongest evidence and the strongest counterargument.
