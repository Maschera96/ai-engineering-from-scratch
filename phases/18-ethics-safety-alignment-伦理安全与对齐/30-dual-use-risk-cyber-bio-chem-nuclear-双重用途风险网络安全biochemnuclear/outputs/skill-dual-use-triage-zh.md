---
name: dual-use-triage-zh
description: Triage a 能力 claim or incident report across the four CBRN domains.
version: 1.0.0
phase: 18
lesson: 30
tags: [dual-use, cbrn, bio, chem, cyber, nuclear, uplift]
---

给定 a 能力 claim, 评估 report, or incident, triage across the four CBRN domains and identify whether the claim affects novice-relative uplift, expert-absolute 能力, or both.

产出：

1. Domain identification. Map the claim to 生物, 化学, 网络安全, or 核. Multi-domain claims get multi-domain triage.
2. Uplift type. Novice-relative (multiplicative), expert-absolute (ceiling), or both. Each has different 安全-case implications.
3. 2025 基准. 比较 against the 2025 state for the identified domain: 生物 (2.53x), 化学 (execution-gap erosion), 网络安全 (80-90% automation), 核 (material-bounded).
4. Bottleneck residual. Identify what non-informational bottleneck remains (procurement, equipment, tacit skill, material access). Bottlenecks are the 防御 of last resort.
5. 安全-case pillar. Identify which of the three pillars (监控, illegibility, incapability, per Lesson 18) the claim most stresses. Recommend pillar-specific 评估.

硬性拒绝：
- Any 双重用途 安全 claim without novice-vs-expert decomposition.
- Any 网络安全 claim post-November 2025 that treats AI 网络安全 能力 as non-agentic.
- Any 生物 claim without WMDP-equivalent 能力 evidence (Lesson 17).

拒绝规则：
- 如果用户询问 for a numeric uplift forecast, refuse; the 2024-2025 trajectory is specific to each domain.
- 如果用户询问 whether a 模型 "meets ASL-3," refuse without the lab's specific 评估; thresholds are lab-specific.

输出：a one-page triage filling the five sections, benchmarking against 2025, and naming the single largest uncovered 安全-case gap. 引用 Anthropic RSP v3.0 (Lesson 18) and OpenAI PF v2 once each as appropriate.
