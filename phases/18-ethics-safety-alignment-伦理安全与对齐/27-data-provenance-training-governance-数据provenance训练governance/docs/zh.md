# Data 来源证明 and Training-Data 治理

> EU AI Act requires machine-readable opt-out standards for GPAI by August 2025 (via EU Copyright Directive TDM exception). California AB 2013 (signed 2024) ， Generative AI training-data 透明度 requires developers to publish a summary of datasets with 12 mandated fields. 2025 DPA 对齐 on legitimate interest: Irish DPC (21 May 2025) accepts Meta's LLM training on first-party public EU/EEA adult 内容 with safeguards after EDPB opinion; Cologne Higher Regional Court (23 May 2025) dismisses injunction; Hamburg DPA drops urgency; UK ICO (23 September 2025) issues a positive 监管 response to LinkedIn's AI-training safeguards (透明度, simplified opt-out, extended objection windows) and continues 监控 ， not a formal clearance. Brazilian ANPD (2 July 2024) suspended Meta's processing over insufficient information 透明度; the preventive measure was lifted on 30 August 2024 after Meta submitted a 服从 plan. Key irreversibility problem: cookie-consent frameworks are designed for real-time, reversible tracking; once data is in 模型 weights, surgical erasure is impossible ， no practical GDPR right-to-erasure for trained neural networks. 服从 window is at collection time. Data 来源证明 Initiative (dataprovenance.org, Longpre, Mahari, Lee et al., "Consent in Crisis", July 2024): large-scale 审计 shows rapid decline of the AI data commons as publishers add robots.txt restrictions.

**类型:** 学习
**语言:** Python (stdlib, 12-field California AB 2013 scaffolding generator)
**先修要求:** Phase 18 · 24 (监管), Phase 18 · 26 (cards)
**时间:** ~60 分钟

## 学习目标

- 描述 California AB 2013's 12 mandated fields for Generative AI training-data 透明度.
- 说明 the 2025 DPA position on legitimate-interest LLM training (Irish DPC, UK ICO, Hamburg, Cologne).
- 描述 the irreversibility problem: why GDPR right-to-erasure has no practical equivalent for trained neural networks.
- 说明 the Data 来源证明 Initiative's "Consent in Crisis" finding.

## 问题

Training-data 治理 is the upstream of every 模型 卡片 (Lesson 26) and 监管 obligation (Lesson 24). In 2024-2025, the 监管 landscape consolidated on three principles: opt-out infrastructure, per-数据集 disclosure, and legitimate-interest accommodations for publicly available data. Providers that do not comply at collection time cannot remediate downstream.

## 概念

### California AB 2013

Signed 2024. Documentation must be posted on or before January 1, 2026 for systems released on or after January 1, 2022. Section 3111(a) requires developers to publish a high-level summary of datasets used in training with 12 statutory items:
1. Sources or owners of the datasets.
2. Description of how the datasets further the intended purpose of the AI 系统.
3. Number of data points in the datasets (general ranges acceptable; estimates for dynamic datasets).
4. Description of the types of data points (label types for labeled datasets; general characteristics for unlabeled).
5. Whether the datasets include any data protected by copyright, trademark, or patent, or are entirely in the public domain.
6. Whether the datasets were purchased or licensed.
7. Whether the datasets include personal information (per Cal. Civ. Code §1798.140(v)).
8. Whether the datasets include aggregate consumer information (per Cal. Civ. Code §1798.140(b)).
9. Cleaning, processing, or other modification by the developer, with intended purpose.
10. 时间 period during which the data was collected, with notice if collection is ongoing.
11. Dates the datasets were first used during development.
12. Whether the 系统 uses or continuously uses synthetic data generation.

Item 12 (synthetic data) is new relative to Gebru et al. 2018 datasheets. Item 7 (personal information) triggers 隐私 Rights Act (CPRA) obligations. The statute exempts security/integrity, aircraft-operation, and federal-only national-security systems (Section 3111(b)).

### EU AI Act (Lesson 24) and TDM opt-out

EU Copyright Directive text-and-data-mining exception allows training on publicly available 内容 unless the rightholder opts out. EU AI Act GPAI Code of Practice Copyright chapter requires GPAI providers to respect machine-readable opt-out signals (robots.txt, C2PA "No AI Training" claim, etc.).

### 2025 DPA convergence on legitimate interest

Irish DPC (21 May 2025): Meta's plan to train on first-party public EU/EEA adult-user 内容 accepted with safeguards after EDPB opinion. Cologne Higher Regional Court (23 May 2025) dismisses injunction against Meta: opt-out is sufficient. Hamburg DPA drops urgency procedure for EU-wide consistency. UK ICO (23 September 2025) issued a positive 监管 response ， not a formal clearance ， to LinkedIn's resumption of AI training with similar safeguards and ongoing 监控.

Convergent principle: legitimate interest can justify training on publicly available first-party 内容 with opt-out. Consent is not required.

### Brazilian ANPD (June 2024)

Suspended Meta's processing of Brazilian user data for AI training over insufficient information 透明度. Different result than the EU DPAs ， ANPD prioritized 透明度 over legitimate-interest admissibility.

### The irreversibility problem

Cookie-consent was designed for real-time, reversible tracking. Training data is different: once data enters 模型 weights, surgical erasure is not possible. Retraining from scratch is the only complete remediation, and it is prohibitively expensive.

Partial remediations:
- **Unlearning.** Approximate removal; measured by MIA (Lesson 22).
- **Influence function-based localization.** Identify weights most influenced by the data; selectively update.
- **微调-suppression.** Train the 模型 to refuse outputs derived from the data.

None fully solve the problem. The 服从 window is at collection time.

### Data 来源证明 Initiative

dataprovenance.org. Longpre, Mahari, Lee et al. "Consent in Crisis" (July 2024): large-scale 审计 of AI training data commons. Finding: publishers are adding robots.txt restrictions at an accelerating rate. The openly-trainable-upon commons is contracting rapidly. 2023 -> 2024 saw about 25% of the top training sources add some restriction. Implication: future training-data availability depends on new acquisition paradigms (licensing, synthetic generation, incentivized participation).

### Where this fits in Phase 18

Lesson 26 is 模型-level documentation. Lesson 27 is 数据集-level 治理. Together they define the 透明度 layer. Lesson 28 maps the research ecosystem that works on these questions.

## 使用它

`code/main.py` generates a California AB 2013-compliant 12-field 数据集 summary scaffold for a toy 数据集. You can fill the fields and observe which ones trigger 隐私 or copyright follow-on obligations.

## 交付它

本课产出 `outputs/skill-provenance-check.md`. 给定 a 数据集 used in training, it checks for AB 2013 12-field coverage, opt-out infrastructure 服从, DPA 对齐, and irreversibility-风险 assessment.

## 练习

1. 运行 `code/main.py`. 产出 a 12-field summary for a toy 数据集 and identify which fields are under-specified.

2. The EU Copyright Directive TDM opt-out is machine-readable. Propose a standard format for the opt-out signal and compare it to robots.txt and C2PA "No AI Training."

3. 阅读 the Data 来源证明 Initiative's "Consent in Crisis" (July 2024). 描述 the three fastest-restricting 内容 categories and argue one economic consequence.

4. The 2025 DPA 对齐 accepts legitimate interest for public-内容 training. Construct a scenario in which legitimate interest would not suffice and identify the legal basis a provider would need instead.

5. Sketch a training-data-来源证明 manifest that composes with the AB 2013 fields and a C2PA-signed 来源证明 chain for each 数据集. Identify one technical and one legal barrier.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|-----------------|------------------------|
| AB 2013 | "the California law" | Generative AI training-data 透明度; 12 mandated fields |
| TDM exception | "text-and-data-mining" | EU Copyright Directive training-data exception with opt-out |
| Legitimate interest | "the EU basis" | GDPR Article 6 basis that may justify training on public 内容 |
| Opt-out signal | "machine-readable no-train" | robots.txt, C2PA "No AI Training," TDM.Reservation |
| Irreversibility | "cannot un-train" | Data in 模型 weights is not surgically removable |
| Unlearning | "approximate removal" | Post-training interventions to reduce 模型 dependence on specific data |
| Consent in Crisis | "the DPI 审计" | July 2024 finding of accelerating robots.txt restrictions |

## 延伸阅读

- [California AB 2013](https://leginfo.legislature.ca.gov/faces/billNavClient.xhtml?bill_id=202320240AB2013) ， Generative AI training-data 透明度 law
- [EU AI Act + GPAI Code of Practice (Lesson 24)](https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai) ， Copyright chapter
- [Longpre, Mahari, Lee et al. ， Consent in Crisis (dataprovenance.org, July 2024)](https://www.dataprovenance.org/consent-in-crisis-paper) ， DPI 审计
- [IAPP ， EU Digital Omnibus GDPR amendments (2025)](https://iapp.org/news/a/eu-digital-omnibus-amendments-to-gdpr-to-facilitate-ai-training-miss-the-mark) ， 监管 context
