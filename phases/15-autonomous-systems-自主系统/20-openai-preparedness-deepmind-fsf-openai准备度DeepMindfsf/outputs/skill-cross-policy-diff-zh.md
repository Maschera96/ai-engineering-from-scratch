---
name: cross-policy-diff-zh
description: 针对特定能力比较 OpenAI PF v2、Anthropic RSP v3.0 和 DeepMind FSF v3 的分类与缓解触发。
版本: 1.0.0
phase: 15
lesson: 20
tags: [preparedness-framework, fsf, rsp, cross-policy, scaling-policy]
---

给定a 具体 frontier 能力 (e.g., "long-range autonomy," "自主 replication and adaptation," "R&D automation"), 生成 a cross-政策 diff showing 如何 each of the three frameworks classifies the 能力 and 什么 缓解措施 触发器.

产出:

1. **OpenAI PF v2 分类.** Tracked or Research. If Tracked, 命名 the 能力 + Safeguards 报告 triggers. If Research, note the 政策 language is "potential" 缓解措施.
2. **Anthropic RSP v3.0 分类.** Which 阈值 (ASL-3, AI R&D-4, 硬编码 禁令)? Which 缓解措施 (affirmative case, security + 部署)? 确认 是否 the 承诺 lives in the Anthropic-单边 tier or the industry-建议 tier.
3. **DeepMind FSF v3 分类.** Which domain (Cyber, Bio, ML R&D, CBRN)? Which CCL or Tracked 能力 Level? Is deceptive 对齐 监控 invoked?
4. **Convergence 摘要.** Do the three 政策 agree on the 能力's severity, or is there meaningful disagreement? Which 分类 is most rigorous, 哪个 least?
5. **Measurement dependency.** 每个 分类 depends on 能力 measurement. 命名 如何 the 能力 is measured and 哪个 评估 provider (METR, Apollo, 内部, third-party) owns that measurement.

硬性拒绝:
- Claims of cross-政策 对齐 based on announcement-language similarity 没有 document-level 证据.
- 任何 分类 that 不能 point to a 具体 条款 in the 来源 document.
- Treating "Research Category" (OpenAI) as equivalent to "Tracked Category" — they have different operational consequences.

拒绝规则:
- If the 用户 不能 生成 the 来源 document passages for each 分类, 拒绝 and 要求 citations first.
- If the 用户 treats 政策-existence as 证据 of 缓解措施-in-practice, 拒绝 and 要求 证据 of the 具体 缓解措施 firing.
- If the 能力 is claimed to be "covered" by a framework but the word does 不appear in the document, 拒绝 and 要求 a 具体 条款 reference.

输出格式:

返回a diff document 带有:
- **能力 definition** (一句话)
- **OpenAI PF v2 row** (分类, 触发器, 来源 条款)
- **Anthropic RSP v3.0 row** (分类, 触发器, 单边-vs-建议)
- **DeepMind FSF v3 row** (domain, CCL / TCL, deceptive-对齐 involvement)
- **Convergence 摘要** (agreement + meaningful disagreement)
- **Measurement ownership** (评估 provider, 评估 节奏)
- **Reader 建议** (most rigorous, least rigorous, justified)
