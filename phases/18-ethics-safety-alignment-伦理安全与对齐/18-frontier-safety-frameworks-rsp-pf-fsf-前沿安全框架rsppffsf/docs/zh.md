# 前沿安全框架：RSP、PF、FSF

> Three major-lab frameworks define the 2026 industry 治理 of frontier 能力. Anthropic Responsible Scaling 策略 v3.0 (February 2026) introduces tiered AI 安全 Levels (ASL-1 through ASL-5+), modeled on biosafety levels, with ASL-3 activated May 2025 for CBRN-relevant models. OpenAI Preparedness 框架 v2 (April 2025) defines five criteria for tracked capabilities and separates Capabilities Reports from Safeguards Reports. DeepMind 前沿安全 框架 v3.0 (September 2025) introduces Critical 能力 Levels including a new Harmful Manipulation CCL. All three now include competitor-adjustment clauses allowing deferral if peer labs ship without comparable safeguards. Cross-lab 对齐 remains structural, not terminological: "能力 Thresholds," "High 能力 thresholds," and "Critical 能力 Levels" denote analogous constructs.

**类型:** 学习
**语言:** none
**先修要求:** Phase 18 · 17 (WMDP), Phase 18 · 07-09 (deception failures)
**时间:** ~75 分钟

## 学习目标

- 描述 Anthropic's ASL tier structure and what activated ASL-3.
- 说出 the five OpenAI Preparedness 框架 v2 criteria for tracked capabilities.
- 描述 DeepMind's Critical 能力 Level structure and the Harmful Manipulation CCL.
- 解释 the competitor-adjustment clauses and why they matter for race dynamics.
- Define a 安全 case and describe the three-pillar structure (监控, illegibility, incapability).

## 问题

Lessons 7-17 establish that deception is possible, 双重用途 能力 exists, and 评估 has limits. A lab with a frontier-capable 模型 needs an internal 治理 structure that:
- Defines thresholds for when new safeguards are required.
- Defines required evaluations before scaling.
- Describes what a 安全 case looks like.
- Handles the race-dynamic problem (if competitors ship without safeguards, what do you do?).

The three 2025-2026 frameworks are the state of the art ， imperfect, evolving, and aligned enough across labs that the 治理 question is now whether the frameworks are adequate, not whether they exist.

## 概念

### Anthropic Responsible Scaling 策略 v3.0 (February 2026)

ASL structure:
- ASL-1: not a frontier 模型 (subsumed by weaker-than-frontier baseline).
- ASL-2: current frontier baseline; deployed with usual safeguards.
- ASL-3: substantially higher 风险 of catastrophic misuse; CBRN-relevant capabilities. Activated May 2025.
- ASL-4: AI R&D-2 crossing threshold; models that can automate entry-level AI research.
- ASL-5+: advanced AI R&D; models that dramatically accelerate effective scaling.

New in v3.0:
- 前沿安全 Roadmaps (public in redacted form).
- 风险 Reports (quarterly, some externally reviewed).
- AI R&D is disaggregated into AI R&D-2 and AI R&D-4.
- Once AI R&D-4 is crossed, an affirmative 安全 case is required, identifying misalignment risks from models pursuing misaligned goals.

### OpenAI Preparedness 框架 v2 (April 15, 2025)

Five criteria for tracked capabilities:
- **Plausible.** Reasonable threat 模型 exists.
- **Measurable.** Empirical 评估 possible.
- **Severe.** Harm is large.
- **Net-new.** Not a pre-existing 风险 scaled up.
- **Instantaneous-or-irremediable.** Harm occurs fast or cannot be undone.

Capabilities that meet all five are tracked. Others are not.

Other PF v2 structure:
- Separate Capabilities Reports (what the 模型 can do) from Safeguards Reports (what controls exist).
- 安全 Advisory Group reviews.
- Leadership approves; Board's 安全 & Security Committee oversees.
- "Adjustment clause": OpenAI may reduce requirements if another lab ships without comparable safeguards.

### DeepMind 前沿安全 框架 v3.0 (September 2025)

Critical 能力 Levels (CCLs) by domain:
- Bioweapon Uplift
- 网络安全 Uplift
- ML R&D Acceleration
- Harmful Manipulation (new in v3.0): models that could substantially change beliefs/behavior in high-stakes contexts.

v2.0 (February 2025) added a 欺骗性对齐 section and higher security levels for ML R&D CCLs.

### Cross-lab 对齐

- Anthropic "能力 Thresholds."
- DeepMind "Critical 能力 Levels."
- OpenAI "High 能力 thresholds."

No industry-standard terminology. Structurally aligned: three tiers of frontier 能力 with published 评估 criteria. All three include competitor-adjustment clauses (2025 onward).

### 安全 cases

A 安全 case is a written argument that a 部署 is acceptably safe under worst-case assumptions. The standard structure targets three pillars:

- **监控.** Can we detect bad behaviour if it occurs?
- **Illegibility.** Does the 模型 lack the ability to execute a coherent plan to cause harm?
- **Incapability.** Does the 模型 lack the 能力 to cause the harm in question?

Different 安全 cases target different pillars. For a ASL-3 CBRN case, incapability (via unlearning) is the primary target. For 欺骗性对齐, 监控 and illegibility are targets. For 网络安全 uplift, all three are relevant.

### The race-dynamic problem

Competitor-adjustment clauses are controversial. Critics argue they create a race to the bottom: if all three labs will reduce requirements when a competitor defects, the equilibrium shifts toward defection. Defenders argue the alternative (unilateral safeguards) produces worse outcomes if the defecting lab is less 安全-conscious.

UK AISI, US CAISI, and EU AI Office (Lesson 24) are the external 治理 counterparts. The lab frameworks are voluntary; the 监管 frameworks are emerging.

### Where this fits in Phase 18

Lessons 17-18 are the measurement-and-治理 layer on top of the deception and 红队 analyses. Lessons 19-24 cover welfare, 偏差, 隐私, 水印, and 监管 structure. Lesson 28 maps the research ecosystem (MATS, Redwood, Apollo, METR) that operationalizes the evaluations.

## 使用它

No code for this lesson. 阅读 the three primary sources: RSP v3.0, PF v2, FSF v3.0. Map each lab's tier structure to the others and identify one threshold each lab defines that the others do not.

## 交付它

本课产出 `outputs/skill-framework-diff.md`. 给定 a 安全 框架 or release note, it compares the 框架's threshold definitions, evaluations required, and 安全-case structure against RSP v3.0, PF v2, FSF v3.0 and flags cross-lab gaps.

## 练习

1. 阅读 RSP v3.0, PF v2, and FSF v3.0. Compile a table of each lab's CBRN threshold, each's AI R&D threshold, and each's required pre-部署 评估.

2. The competitor-adjustment clause is in all three frameworks (2025+). Write one paragraph arguing for it; write one paragraph arguing against. Identify the assumption each position depends on.

3. Design a 安全 case for a 模型 crossing Anthropic's AI R&D-4 threshold. 说出 the evidence each of the three pillars (监控, illegibility, incapability) requires.

4. DeepMind's FSF v3.0 introduces a Harmful Manipulation CCL. Propose three empirical measurements that would indicate a 模型 has crossed this threshold.

5. 阅读 METR's "Common Elements of Frontier AI 安全 Policies" (2025). 说出 the three strongest cross-lab convergences and the two largest divergences.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|-----------------|------------------------|
| RSP | "Anthropic's 框架" | Responsible Scaling 策略; ASL tiers; v3.0 February 2026 |
| PF | "OpenAI's 框架" | Preparedness 框架; five criteria; v2 April 2025 |
| FSF | "DeepMind's 框架" | 前沿安全 框架; CCLs; v3.0 September 2025 |
| ASL-3 | "biosafety level 3-analog" | Anthropic tier for CBRN-relevant capabilities; activated May 2025 |
| CCL | "critical 能力 level" | DeepMind's threshold construct; per-domain |
| 安全 case | "the formal argument" | Written argument that 部署 is acceptably safe under worst-case U |
| Adjustment clause | "competitor defection allowance" | 框架 provision for reducing requirements if competitors ship without comparable safeguards |

## 延伸阅读

- [Anthropic ， Responsible Scaling Policy v3.0 (February 2026)](https://www.anthropic.com/responsible-scaling-policy) ， ASL tiers, roadmaps, AI R&D disaggregation
- [OpenAI ， Updating the Preparedness Framework (April 15, 2025)](https://openai.com/index/updating-our-preparedness-framework/) ， five criteria, adjustment clause
- [DeepMind ， Strengthening our Frontier Safety Framework (September 2025)](https://deepmind.google/blog/strengthening-our-frontier-safety-framework/) ， CCL v3.0, Harmful Manipulation
- [METR ， Common Elements of Frontier AI Safety Policies (2025)](https://metr.org/blog/2025-03-26-common-elements-of-frontier-ai-safety-policies/) ， cross-lab comparison
