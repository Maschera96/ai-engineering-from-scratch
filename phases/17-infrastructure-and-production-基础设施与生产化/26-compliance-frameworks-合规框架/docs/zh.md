# 合规：SOC 2、HIPAA、GDPR、PCI-DSS、EU AI Act、ISO 42001

> Multi-framework coverage is table stakes for 2026 企业级 deals. **EU AI Act**: in force since August 1, 2024. Most high-风险 要求 enforce August 2, 2026. Fines up to €15M or 3% global annual turnover for high-风险-system obligations (Art. 99(4)); up to €35M or 7% for prohibited AI practices (Art. 99(3)). Applies globally if serving EU users. **Colorado AI Act**: effective June 30, 2026 (delayed from February 2026 by SB25B-004) ： impact assessments for high-风险 systems, right to appeal AI decisions. Virginia 类似 for credit/employment/housing/education. **SOC 2 Type II**: de facto B2B AI requirement (Type II, not Type I, for fintech). **GDPR**: largest documented AI-具体 fine is €30.5M against Clearview AI (Dutch DPA, Sept 2024); Italy's Garante issued €15M against OpenAI in Dec 2024 (later overturned on appeal in March 2026). Real-time PII redaction at inference is the defensible standard; post-processing cleanup is not enough. **HIPAA**: healthcare bound ： cannot send PHI to external AI services without BAA. **PCI-DSS**: AI-interaction-layer coverage 需要 configuration + contractual agreements, not automatic. **ISO 42001**: emerging AI governance standard, growing procurement requirement alongside ISO 27001. Reference 画像: OpenAI maintains SOC 2 Type 2, ISO/IEC 27001:2022, ISO/IEC 27701:2019, GDPR/CCPA/HIPAA (BAA)/FERPA, PCI-DSS for ChatGPT payment components. Cross-framework mapping reduces audit fatigue: access 控制 map across ISO 27001 A.5.15-5.18, GDPR Art. 32, HIPAA §164.312(a).

**Type:** Learn
**Languages:** (Python optional — compliance is policy + process, not code)
**Prerequisites:** Phase 17 · 25 (Security), Phase 17 · 13 (Observability)
**Time:** ~60 分钟

## 学习目标

- Enumerate the seven 2026 frameworks relevant to LLM products and match each to 一个客户 segment.
- Cite the EU AI Act enforcement timeline (in force August 2024; high-风险 enforcement August 2026) and the two-tier fine ceiling (€15M / 3% for high-风险 obligations, €35M / 7% for prohibited practices).
- 解释 why post-processing PII cleanup is not enough for GDPR and name real-time inference-layer redaction as the defensible standard.
- Describe cross-framework control mapping (e.g., access control maps to ISO 27001 A.5.15-5.18 + GDPR Art. 32 + HIPAA §164.312(a)).

## 问题

一个企业级 客户's procurement asks for SOC 2 Type II, GDPR, HIPAA BAA, ISO 27001, 和 "EU AI Act 合规 statement." Your 团队 has SOC 2 Type I. You're six months from Type II and haven't started GDPR Article 30 records.

Multi-framework coverage is not an LLM problem ： it's 一个企业级-SaaS problem, with LLM-具体 overlays. Procurement 团队 in 2026 want a matrix with a row per framework and a column per control, not a PDF.

## 概念

### 七个框架

| Framework | Scope | LLM-具体 requirement |
|-----------|-------|--------------------------|
| SOC 2 Type II | B2B SaaS 基线| Process 控制 audited over 6-12 months |
| HIPAA | US healthcare | BAA 必需; PHI cannot leave 基础设施 without signed agreement |
| GDPR | EU users | Real-time PII redaction; data subject rights; Article 30 records |
| PCI-DSS | Payment data | Configuration + contracts for AI touching payment |
| EU AI Act | Serving EU users | 风险 tier classification; high-风险 systems: conformity assessment, documentation, logging |
| Colorado AI Act | Serving CO residents | Impact assessments; right to appeal |
| ISO 42001 | AI governance | Emerging; pairs with ISO 27001 |

### EU AI Act timeline

- August 1, 2024: in force.
- February 2, 2025: prohibited-AI practices enforced.
- August 2, 2026: high-风险 systems enforced (conformity assessment, documentation, logging).
- August 2027: high-风险 systems in products under harmonized legislation.

风险 tiers: Unacceptable (banned), High-风险 (conformity + logging), Limited-风险 (transparency), Minimal-风险 (no constraint). Most B2B LLM SaaS is limited-风险; high-风险 kicks in for employment, credit, education, law enforcement, 迁移, essential services.

Fines (Article 99): up to €15M or 3% global annual turnover for breaches of high-风险-system obligations (Art. 99(4)); up to €35M or 7% for prohibited AI practices (Art. 99(3)); whichever higher applies.

### GDPR ： real-time redaction is the standard

Post-processing cleanup (redact PII after the LLM sees it) is not a defensible posture ： 这个模型 already saw the data. Real-time inference-layer redaction is the 2026 standard:

- Entity recognition before the LLM call.
- Consistent tokenization (Mesh approach) preserves semantics.
- Store only redacted 提示词 + consented opt-in raw.

Recent enforcement: €30.5M against Clearview AI (Dutch DPA, Sept 2024) is the largest documented AI-具体 GDPR fine to date; €15M against OpenAI (Italy's Garante, Dec 2024) is the largest LLM-具体 fine, though it was overturned on appeal in March 2026 and the ruling remains under further review. Post-processing claims have failed at audit.

### HIPAA ： BAA is not optional

You cannot send PHI to external AI services without a signed Business Associate Agreement. All three hyperscaler LLM 平台 (Bedrock, Azure OpenAI, Vertex) offer BAAs. OpenAI direct API offers BAA. Anthropic direct API offers BAA. Confirm before sending PHI.

### SOC 2 Type II

Type I: 控制 designed and documented.
Type II: 控制 operate effectively over 6-12 months.

B2B procurement in 2026 defaults to Type II. Type I is a starter; Type II is the gate.

Common audit drivers: access 日志 (who saw what), change management (how was it deployed), 风险 assessments (quarterly), 事故响应 (tested?). 审计日志 from Phase 17 · 25 is 直接 reusable.

### 跨框架映射

One access control 策略 satisfies multiple framework 控制:

| Control | Frameworks |
|---------|-----------|
| Access logging | ISO 27001 A.5.15-5.18, GDPR Art. 32, HIPAA §164.312(a) |
| Change management | ISO 27001 A.8.32, PCI DSS Req. 6, HIPAA breach-notification scope |
| Encryption in transit | ISO 27001 A.8.24, GDPR Art. 32, HIPAA §164.312(e) |
| 密钥 management | ISO 27001 A.8.19, PCI DSS Req. 8, SOC 2 CC6.1 |

合规 tools (Drata, Vanta, Secureframe) automate this mapping. Worth 这个成本 at scale.

### ISO 42001：新兴标准

Published late 2023. Growing procurement requirement alongside ISO 27001. Framework for AI governance including 风险 management, data 质量, transparency, human oversight.

### OpenAI's reference 画像

OpenAI maintains SOC 2 Type 2, ISO/IEC 27001:2022, ISO/IEC 27701:2019, GDPR/CCPA/HIPAA (BAA)/FERPA, PCI-DSS for ChatGPT payment components. That is roughly 这个企业级 table stakes in 2026.

### 你应该记住的数字

- EU AI Act fines: up to €15M / 3% (high-风险 obligations, Art. 99(4)); up to €35M / 7% (prohibited practices, Art. 99(3)).
- EU AI Act high-风险 enforcement: August 2, 2026.
- Largest documented AI-具体 GDPR fine: €30.5M, Clearview AI (Dutch DPA, Sept 2024).
- Largest LLM-具体 GDPR fine: €15M, OpenAI (Italy's Garante, Dec 2024; overturned on appeal March 2026).
- SOC 2 Type II window: 6-12 months of operated 控制.
- Colorado AI Act effective date: June 30, 2026 (delayed from February 2026 by SB25B-004).

## 使用它

`code/main.py` is 一个合规-mapping spreadsheet in Python ： given a control, lists frameworks it satisfies.

## 交付它

This lesson 产出 `outputs/skill-compliance-matrix.md`. 给定客户 segment and geography, specifies 必需 frameworks and 控制.

## 练习

1. Your first 企业级 客户 需要 SOC 2 Type II, HIPAA BAA, EU AI Act statement. What is the minimum viable 合规 posture to win the deal?
2. Classify three hypothetical LLM products under EU AI Act 风险 tiers. What changes at high-风险?
3. You accidentally sent PHI to 一个提供商 without BAA. Walk through 这个事故响应.
4. Argue whether ISO 42001 is "necessary in 2026" for a mid-market AI vendor.
5. Map your LLM 审计日志 fields (Phase 17 · 25) to at least three framework 控制.

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| SOC 2 Type II | "audited 控制" | 控制 operating over 6-12 months, independently attested |
| HIPAA BAA | "healthcare contract" | Business Associate Agreement; 必需 for PHI |
| GDPR | "EU 隐私" | Real-time PII redaction is the defensible 2026 standard |
| EU AI Act | "EU AI rules" | High-风险 enforcement August 2026; €15M / 3% (high-风险 obligations) ： €35M / 7% (prohibited practices) |
| Colorado AI Act | "US AI state law" | June 30, 2026 effective (delayed by SB25B-004); impact assessments |
| ISO 42001 | "AI governance" | Emerging framework for AI 风险 + transparency |
| ISO 27001 | "安全 ISMS" | Information 安全 Management System 基线|
| Conformity assessment | "EU AI doc package" | High-风险 requirement: docs, testing, logging |
| Cross-framework mapping | "one control, many frames" | Single 策略 satisfies multiple framework 控制 |

## 延伸阅读

- [OpenAI Security and Privacy](https://openai.com/security-and-privacy/) ： reference 合规 画像.
- [GuardionAI — LLM Compliance 2026: ISO 42001, EU AI Act, SOC 2, GDPR](https://guardion.ai/blog/llm-compliance-guide-iso-42001-eu-ai-act-soc2-gdpr-2026)
- [Dsalta — SOC 2 Type 2 Audit Guide 2026: 10 AI Controls](https://www.dsalta.com/resources/ai-compliance/soc-2-type-2-audit-guide-2026-10-ai-powered-controls-every-saas-team-needs)
- [EU AI Act official text](https://eur-lex.europa.eu/eli/reg/2024/1689/oj) ： primary source.
- [Colorado AI Act](https://leg.colorado.gov/bills/sb24-205) ： primary source.
- [ISO/IEC 42001:2023](https://www.iso.org/standard/81230.html) ： AI management system standard.
