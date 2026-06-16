# 模型, 系统, and 数据集 Cards

> Three documentation formats structure AI 透明度. 模型 Cards (Mitchell et al. 2019) ， nutrition labels for models: training data, quantitative disaggregated analyses, ethical considerations, caveats; only 0.3% of Hugging Face 模型 cards document ethical considerations (Oreamuno et al. 2023). Datasheets for Datasets (Gebru et al. 2018, CACM) ， motivation, composition, collection process, labeling, distribution, maintenance; electronics-数据说明书 analogy. Data Cards (Pushkarna et al., Google 2022) ， modular layered detail (telescopic, periscopic, microscopic) as boundary objects for diverse readers. 2024-2025 developments: automated generation via LLMs (CardGen, Liu et al. 2024); 模型-卡片 detail correlates with up to 29% download increase on HF (Liang et al. 2024); verifiable attestations (Laminator, Duddu et al. 2024); sustainability reporting additions for carbon/water (Jouneaux et al. July 2025); EU/ISO 监管 cards emerging. 系统 Cards (Sidhpurwala 2024; Meta 系统-level 透明度; "Blueprints of Trust" arXiv:2509.20394) ， end-to-end AI 系统 documentation covering security capabilities, prompt-injection protection, data-exfiltration detection, 对齐 with human values.

**类型:** 构建
**语言:** Python (stdlib, 模型-卡片 + 数据说明书 + 系统-卡片 generator)
**先修要求:** Phase 18 · 18 (安全 frameworks), Phase 18 · 24 (监管)
**时间:** ~60 分钟

## 学习目标

- 描述 the original Mitchell et al. 2019 模型 卡片 and the Gebru et al. 2018 数据说明书.
- 描述 Data Cards' telescopic/periscopic/microscopic layering.
- 描述 系统 Cards and their end-to-end coverage.
- 说明 three 2024-2025 developments (automated generation, verifiable attestations, sustainability reporting).

## 问题

监管 frameworks (Lesson 24) and lab 安全 policies (Lesson 18) both require documentation. Documentation formats evolved from 模型-specific (模型 cards) to 数据集-specific (datasheets) to 系统-specific (系统 cards). Each addresses a different scope of 透明度. The 2024-2025 automation and verifiable-attestation work addresses the long-standing adoption problem.

## 概念

### 模型 Cards (Mitchell et al. 2019)

Sections:
- 模型 details.
- Intended use.
- Factors (relevant demographic or environmental factors for 评估).
- Metrics.
- 评估 data.
- Training data.
- Quantitative analyses (disaggregated by factors).
- Ethical considerations.
- Caveats and recommendations.

Adoption problem: Oreamuno et al. 2023 审计 of Hugging Face 模型 cards found only 0.3% document ethical considerations.

### Datasheets for Datasets (Gebru et al. 2018)

Electronics-数据说明书 analogy. Sections:
- Motivation (why was the 数据集 created).
- Composition (what is in it).
- Collection process (how was it assembled).
- Labeling (if applicable).
- Uses (intended, prohibited, risks).
- Distribution.
- Maintenance.

Published in CACM 2021. The 数据说明书 is the upstream documentation; the 模型 卡片 depends on the 数据说明书 being accurate.

### Data Cards (Pushkarna et al., Google 2022)

Modular layered detail. Three zoom levels:
- **Telescopic.** High-level summary for non-experts.
- **Periscopic.** Middle-level overview for ML practitioners.
- **Microscopic.** Detailed feature-level documentation for auditors.

Boundary-object framing: different readers extract different information from the same document.

### 系统 Cards

Scope: end-to-end AI 系统 including 模型 + 安全 stack + 部署 context. Sections typically include:
- Security capabilities.
- Prompt-injection protection.
- Data-exfiltration detection.
- 对齐 with stated human values.
- Incident response.

Sidhpurwala 2024 and Meta 系统-level 透明度 work. "Blueprints of Trust" (arXiv:2509.20394) formalizes the 系统 卡片 as the 部署-layer complement to 模型 Cards.

### 2024-2025 developments

- **CardGen (Liu et al. 2024).** Automated 模型-卡片 generation via LLMs; reports higher objectivity than many human-authored cards on the standardized Mitchell 2019 fields.
- **Download correlation (Liang et al. 2024).** Detailed 模型 cards correlate with up to 29% higher download rates on HF ， adoption pressure is now market-driven, not only 服从-driven.
- **Laminator (Duddu et al. 2024).** Verifiable attestations via hardware TEE / cryptographic signatures ， allows the 模型 卡片 to carry a proof-of-claim, not just a claim.
- **Sustainability (Jouneaux et al. July 2025).** Additions for carbon, water, and compute-energy footprint; emerging ISO standards.
- **监管 cards.** EU AI Act (Lesson 24) GPAI Code of Practice 透明度 chapter requires 模型 cards as a 服从 artifact.

### Where this fits in Phase 18

Lessons 24-25 are 监管 and CVE layers. Lesson 26 is the documentation layer. Lesson 27 is training-data 治理, which is the 数据说明书's upstream. Lesson 28 is the research ecosystem that produces evaluations referenced in cards.

## 使用它

`code/main.py` generates a minimal 模型 卡片, 数据说明书, and 系统 卡片 for a toy 部署. Each follows the canonical section structure. You can inspect the format and compare the three scopes.

## 交付它

本课产出 `outputs/skill-card-audit.md`. 给定 a 模型 卡片, 数据说明书, or 系统 卡片, it audits section coverage, numerical disaggregation, and whether verifiable attestations are present.

## 练习

1. 运行 `code/main.py`. Inspect the generated cards. Identify sections that are weak (placeholder-only) and specify what evidence would strengthen them.

2. Extend the 模型 卡片 with a quantitative disaggregated analysis across two demographic groups (Lesson 20).

3. 阅读 Oreamuno et al. 2023 on the 0.3% adoption rate. Propose one structural change to the 模型 卡片 specification that would increase ethical-considerations adoption.

4. Laminator (Duddu et al. 2024) uses TEEs for verifiable attestations. Design a 模型-卡片 field that carries a cryptographic attestation of an 评估 result and describe the verifier's role.

5. Write a 系统 卡片 (系统 卡片, not 模型 卡片) for one of your past projects or a hypothetical 部署. Identify the highest-value section for third-party auditors.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|-----------------|------------------------|
| 模型 卡片 | "the Mitchell 卡片" | Mitchell et al. 2019 standard documentation for ML models |
| 数据说明书 | "the Gebru 数据说明书" | Gebru et al. 2018 standard documentation for datasets |
| Data 卡片 | "the Pushkarna 卡片" | Google 2022 modular layered data documentation |
| 系统 卡片 | "the 部署 卡片" | End-to-end AI 系统 documentation including 安全 stack |
| Boundary object | "different readers, one doc" | Data Cards framing: same document serves diverse audiences |
| Verifiable attestation | "the Laminator attestation" | Cryptographic or TEE proof attached to a documentation claim |
| Sustainability field | "carbon / water footprint" | Emerging 2025 addition for environmental accounting |

## 延伸阅读

- [Mitchell et al. ， Model Cards for Model Reporting (arXiv:1810.03993, FAT* 2019)](https://arxiv.org/abs/1810.03993) ， the canonical 模型 卡片
- [Gebru et al. ， Datasheets for Datasets (CACM 2021, arXiv:1803.09010)](https://arxiv.org/abs/1803.09010) ， 数据说明书 paper
- [Pushkarna et al. ， Data Cards (Google 2022)](https://arxiv.org/abs/2204.01075) ， layered data documentation
- [Sidhpurwala et al. ， Blueprints of Trust (arXiv:2509.20394)](https://arxiv.org/abs/2509.20394) ， 系统 卡片 formalization
