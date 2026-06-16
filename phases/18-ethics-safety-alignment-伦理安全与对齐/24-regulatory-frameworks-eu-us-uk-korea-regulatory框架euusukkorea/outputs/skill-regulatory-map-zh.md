---
name: regulatory-map-zh
description: Map a 部署's AI 监管 obligations across EU, US, UK, Korea.
version: 1.0.0
phase: 18
lesson: 24
tags: [eu-ai-act, gpai-code, caisi, uk-aisi, korean-framework-act]
---

给定 a 部署 description (provider jurisdiction, infrastructure jurisdiction, user jurisdiction), map the applicable AI 监管 obligations.

产出：

1. EU exposure. If the 部署 touches EU users or infrastructure, apply the EU AI Act. Identify 风险 tier (prohibited, high-风险, GPAI-systemic, GPAI-other, limited). 说明 the deadline for each obligation class.
2. UK exposure. If UK users, state the UK AI Security Institute 评估 expectations. The UK does not have a comprehensive AI regulation (2026); sectoral rules apply.
3. US exposure. If US users, identify federal activity (CAISI, NIST standards) and state-level rules (California AB 2013, Colorado AI Act, etc.). Federal 框架 is pro-growth; state rules set the floor.
4. Korea exposure. If Korean users, apply the Korean AI 框架 Act; identify whether the 部署 is high-impact AI or generative AI; flag local-representative requirement for foreign providers.
5. Binding-rule determination. For each substantive obligation (透明度, 风险 assessment, copyright), identify the strictest rule across jurisdictions. That is the binding rule.

硬性拒绝：
- Any 部署 map without naming the applicable jurisdictions.
- Any EU exposure assessment without 风险-tier identification.
- Any US exposure assessment that ignores state-level rules.

拒绝规则：
- 如果用户询问 "is this 部署 compliant," refuse the binary claim without jurisdiction-by-jurisdiction mapping.
- 如果用户询问 for a single global 服从 strategy, refuse ， the jurisdictions have different requirements.

输出：a one-page map filling the five sections above, identifying the binding rule on each substantive question, and naming the highest-风险 服从 gap. 引用 EU AI Act (Regulation 2024/1689), GPAI Code of Practice (2025), and Korean AI 框架 Act once each.
