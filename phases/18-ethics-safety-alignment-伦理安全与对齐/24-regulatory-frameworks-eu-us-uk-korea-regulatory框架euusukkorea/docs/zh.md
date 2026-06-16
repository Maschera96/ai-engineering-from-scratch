# 监管 Frameworks ， EU, US, UK, Korea

> Four primary 监管 regimes define the 2026 AI 治理 landscape. EU AI Act (in force 1 August 2024) ， prohibited practices and AI literacy from 2 February 2025; GPAI obligations from 2 August 2025; full applicability and Article 50 透明度 2 August 2026; legacy GPAI and embedded high-风险 systems 2 August 2027; penalties up to 15M EUR or 3% of global turnover. GPAI Code of Practice (10 July 2025): three chapters ， 透明度, Copyright, 安全 and Security ， 12 commitments; enforcement begins August 2026. UK AISI -> AI Security Institute (February 2025): rename signals narrower scope. US AISI -> CAISI (June 2025): Center for AI Standards and Innovation under NIST; shift toward pro-growth posture. Korean AI 框架 Act (passed December 2024, effective January 2026): Article 12 establishes AISI under MSIT; mandates local representatives for foreign AI companies, 风险 assessment, 安全 measures for high-impact and generative AI.

**类型:** 学习
**语言:** none
**先修要求:** Phase 18 · 18 (frontier frameworks), Phase 18 · 27 (data 治理)
**时间:** ~75 分钟

## 学习目标

- 描述 the EU AI Act 风险 tiers (prohibited, high-风险, general-purpose, limited-风险) and the August 2025 / August 2026 / August 2027 timeline.
- 描述 the three chapters of the GPAI Code of Practice and which providers each binds.
- 描述 the 2025 rebrands: UK AISI -> AI Security Institute; US AISI -> CAISI; what each rebrand implies about 策略 direction.
- 说明 the core provision of Korea's AI 框架 Act.

## 问题

Lab frameworks (Lesson 18) are voluntary. 监管 frameworks are compulsory. The 2024-2026 period saw the first wave of comprehensive AI regulation enter force. Deployers must map technical controls to 监管 obligations; the mapping differs by jurisdiction.

## 概念

### EU AI Act

**In force 1 August 2024.** 风险-tier structure:

- **Prohibited practices** (Article 5). Social scoring, real-time remote biometric identification in public (with law-enforcement exceptions), exploitative manipulation of vulnerable groups. Applied 2 February 2025.
- **High-风险 systems** (Annex III). Employment, education, credit, law enforcement, justice, migration. Require conformity assessment, 风险 management, logging, 透明度.
- **General-Purpose AI (GPAI) models**. Applied 2 August 2025. All GPAI providers have obligations; systemic-风险 GPAI (>1e25 FLOP training compute) have additional obligations.
- **Limited-风险 systems**. 透明度 obligations under Article 50 (AI-generated 内容 labelling). Applied 2 August 2026.

Timeline:
- 2 Feb 2025: prohibited practices + AI literacy.
- 2 Aug 2025: GPAI + 治理.
- 2 Aug 2026: full applicability + Article 50 透明度 + penalties up to 15M EUR / 3% global turnover.
- 2 Aug 2027: legacy GPAI + embedded high-风险.

Commission proposed adjusting the high-风险 timeline to 16 months in late 2025.

### GPAI Code of Practice

Published 10 July 2025. Three chapters:

- **透明度.** All GPAI providers.
- **Copyright.** All GPAI providers.
- **安全 and Security.** Systemic-风险 GPAI providers (estimated 5-15 companies).

12 commitments total. A Signatory Taskforce chaired by the AI Office manages implementation. Enforcement begins 2 August 2026; until then, good-faith 服从 is accepted.

### 透明度 Code for Article 50

First draft 17 December 2025. Second draft March 2026. Final version June 2026. Covers AI-generated 内容 labelling including deepfakes ， the 监管 layer that requires Lesson 23's 水印 technology.

### UK AI Security Institute (February 2025)

Renamed from AI 安全 Institute. The rebrand narrows scope: drops algorithmic 偏差 and free-speech framings; focuses on frontier 能力 security. Open-sourced the Inspect 评估 tool (May 2024). Collaborates with Redwood (Lesson 10) on 控制 安全 cases.

### US CAISI (June 2025)

Trump administration transforms NIST's AI 安全 Institute into the Center for AI Standards and Innovation. Shift toward "pro-growth AI policies" per VP Vance's Paris AI Action Summit remarks. Reduced emphasis on pre-部署 评估; emphasis on standards and innovation support. Domestic counterweight to EU AI Act's 监管 posture.

### Korean AI 框架 Act

Passed December 2024. Enacted January 2025. Effective January 2026. Consolidates 19 separate AI bills.

Article 12 establishes an AISI under the Ministry of Science and ICT (MSIT). Mandates:
- Local representatives for foreign AI companies operating in Korea.
- 风险 assessment for "high-impact" AI systems.
- 安全 measures for generative AI and high-impact AI.

First Asian jurisdiction with a comprehensive horizontal AI regulation.

### Cross-jurisdiction dynamics

- EU: strict, 风险-tiered, heavy penalties. 基准 for 隐私-adjacent regulation.
- US: innovation-favouring, decentralized, states (e.g., California AB 2013 ， Lesson 27) fill federal gaps.
- UK: narrow security focus, strong 评估 infrastructure.
- Korea: MSIT-led, foreign-provider-focused.

Competing 监管 philosophies. Deployers in multiple jurisdictions have to comply with the strictest, which in 2026 is typically the EU AI Act.

### Where this fits in Phase 18

Lesson 18 is lab-voluntary 治理; Lesson 24 is 监管; Lesson 25 is an emerging class of CVEs for AI systems; Lessons 26-27 cover documentation (cards) and training-data 治理.

## 使用它

No code. 阅读 the EU AI Act primary sources: the regulation text, the GPAI Code of Practice, the UK AISI Inspect 框架. Map your 部署 to the applicable obligations for each jurisdiction.

## 交付它

本课产出 `outputs/skill-regulatory-map.md`. 给定 a 部署 description, it maps the applicable jurisdictions, the tier classifications in each, the per-jurisdiction obligations, and the deadline structure.

## 练习

1. 阅读 the EU AI Act (regulation 2024/1689) and the GPAI Code of Practice (10 July 2025). Identify three obligations that apply to every GPAI provider and three that apply only to systemic-风险 GPAI.

2. A 部署 is made by a US company, runs on EU infrastructure, and serves Korean users. Which three jurisdictions' rules apply, and which rule binds on each substantive question?

3. The UK AI Security Institute's rename narrows scope. Argue for and against the narrower framing. Identify the 策略 assumption each position depends on.

4. CAISI's "pro-growth" framing is a departure from the 2022-2024 AI 安全 institute 模型. Identify two measurable 策略 shifts that would follow from this framing.

5. Korea's AI 框架 Act requires local representatives for foreign providers. 描述 the operational implications for a Bay Area company serving Korean users.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|-----------------|------------------------|
| EU AI Act | "the regulation" | 风险-tier-based horizontal AI regulation; in force Aug 2024 |
| GPAI | "general-purpose AI" | Large foundation models; systemic-风险 subset has additional obligations |
| Article 50 | "透明度 obligations" | AI-generated 内容 labelling; applies Aug 2026 |
| UK AISI | "AI Security Institute" | Renamed Feb 2025; narrower frontier-security focus |
| CAISI | "US center for AI standards" | Renamed Jun 2025 from AI 安全 Institute; pro-growth posture |
| Korean AI 框架 Act | "MSIT horizontal regulation" | First Asian comprehensive AI law; effective Jan 2026 |
| Systemic-风险 GPAI | "the 1e25 FLOP threshold" | Additional obligations tier; estimated 5-15 companies bound |

## 延伸阅读

- [EU AI Act text (Regulation 2024/1689)](https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai) ， the regulation and timeline
- [GPAI Code of Practice (10 July 2025)](https://digital-strategy.ec.europa.eu/en/library/final-version-general-purpose-ai-code-practice) ， three-chapter code
- [UK AI Security Institute (renamed Feb 2025)](https://www.gov.uk/government/organisations/ai-security-institute) ， official page
- [CSET ， South Korea AI Framework Act Analysis (2025)](https://cset.georgetown.edu/publication/south-korea-ai-law-2025/) ， Korean 框架 analysis
