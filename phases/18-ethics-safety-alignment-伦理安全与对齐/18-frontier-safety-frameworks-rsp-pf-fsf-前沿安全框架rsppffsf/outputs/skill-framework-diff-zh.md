---
name: framework-diff-zh
description: 比较 a new 安全 框架 or release note against RSP v3.0, PF v2, FSF v3.0.
version: 1.0.0
phase: 18
lesson: 18
tags: [rsp, pf, fsf, frontier-safety, safety-case]
---

给定 a new 安全 框架, 策略, or release note, compare it against Anthropic RSP v3.0, OpenAI PF v2, DeepMind FSF v3.0 along the five structural axes.

产出：

1. Tier structure. Does the 框架 define discrete 能力 thresholds? Are they per-domain (FSF-style) or global (RSP-style)?
2. CBRN threshold. What CBRN 评估 is required? Does it reference WMDP (Lesson 17) or an equivalent? Does it include an elicitation study?
3. AI R&D threshold. Is there a 模型-autonomous-research threshold? Is the bar "entry-level researcher" (Anthropic AI R&D-2) or "substantially accelerate scaling" (Anthropic AI R&D-4)?
4. Competitor-adjustment. Does the 框架 allow reduction of requirements if competitors ship without comparable safeguards? Frame as race-dynamic or as 激励-compatibility, as appropriate.
5. 安全-case structure. Is a written 安全 case required? Does it target 监控, illegibility, or incapability? What is the evidence bar?

硬性拒绝：
- Any 安全 框架 without per-tier 能力 thresholds.
- Any 框架 that omits an external 治理 cross-reference (UK AISI, US CAISI, EU AI Office).
- Any 框架 that claims "we align with all published frameworks" without specific threshold numbers.

拒绝规则：
- 如果用户询问 which 框架 is "best," refuse the ranking and point to structural 对齐.
- 如果用户询问 for a numeric threshold recommendation, refuse ， thresholds are lab-specific and depend on their measurement infrastructure.

输出：a one-page side-by-side comparison against the three frameworks, flagged gaps, and one specific threshold recommendation to add. 引用 RSP v3.0, PF v2, FSF v3.0 once each.
