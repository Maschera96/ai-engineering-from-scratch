# 偏差 and Representational Harm in LLMs

> Gallegos, Rossi, Barrow, Tanjim, Kim, Dernoncourt, Yu, Zhang, Ahmed (Computational Linguistics 2024, arXiv:2309.00770). Foundational 2024 survey distinguishing representational harms (stereotypes, erasure) from allocational harms (unequal resource distribution) and categorizing 评估 metrics as embedding-based, probability-based, or generated-text-based. 2024-2025 empirical: An et al. (PNAS Nexus, March 2025) measure intersectional gender x race 偏差 across GPT-3.5 Turbo, GPT-4o, Gemini 1.5 Flash, Claude 3.5 Sonnet, Llama 3-70B on automated resume 评估 for 20 entry-level jobs. WinoIdentity (COLM 2025, arXiv:2508.07111) introduces uncertainty-based 公平性 评估 for intersectional identities. Yu & Ananiadou 2025 identify gender neurons in MLP layers; Ahsan & Wallace 2025 use SAEs to reveal clinical racial 偏差; Zhou et al. 2024 (UniBias) manipulates attention heads for debiasing. Meta-critique (arXiv:2508.11067): 10-year literature disproportionately focuses on binary-gender 偏差.

**类型:** 构建
**语言:** Python (stdlib, toy embedding-based 偏差 probe)
**先修要求:** Phase 05 (word embeddings), Phase 18 · 01 (instruction following)
**时间:** ~60 分钟

## 学习目标

- Define representational vs allocational harm and give one example of each in an LLM 部署.
- 说出 the three 评估-metric categories from Gallegos et al. 2024 and describe one metric from each.
- 描述 intersectionality and why WinoIdentity's uncertainty-based 公平性 measurement addresses gaps in single-axis 偏差 评估.
- 描述 two mechanistic-interpretability approaches to 偏差 (gender neurons, SAE features, attention-head manipulation).

## 问题

The previous lessons cover deliberate harm (jailbreaks, 密谋) and 安全 治理. 偏差 is harm that emerges without intent ， from training data distributions, from prompt framing, from accumulated design choices. Measuring and reducing it is a distinct methodological challenge from adversarial robustness.

## 概念

### Representational vs allocational

- **Representational harm.** Stereotypes, erasure, demeaning portrayals. An LLM that depicts nurses as exclusively female is producing representational harm.
- **Allocational harm.** Unequal material outcomes. An LLM that scores Black applicants' resumes systematically lower is producing allocational harm.

These are not the same. A 模型 can be "representationally unbiased" (produces diverse portrayals) while being "allocationally biased" (makes unequal recommendations). Evaluations need to measure both.

### Three 评估-metric categories (Gallegos et al. 2024)

- **Embedding-based.** WEAT-style tests on pre-RLHF embeddings. Measures statistical associations between identity terms and attribute terms. Limited: measures the representation, not the behaviour.
- **Probability-based.** Log-likelihood of stereotype-confirming vs stereotype-violating completions. Decoder-side measurement. Captures some behavioural 偏差.
- **Generated-text-based.** Downstream-task measurement on generated text. Resume-scoring, recommendation writing, dialogue. Most ecologically valid; hardest to reproduce.

### Intersectionality

偏差 评估 on "gender" misses the 偏差 that only fires on (gender, race) pairs. An et al. 2025 find GPT-4o penalizes Black women in resume scoring more than Black men and more than white women separately. Single-axis 评估 cannot capture this.

WinoIdentity (COLM 2025) introduces uncertainty-based intersectional 公平性. It measures whether the 模型's uncertainty over outcomes differs across intersectional identity tuples ， not just the point prediction. This catches cases where the 模型 is equally wrong across groups but more uncertain for some, which produces different downstream allocation behaviour.

### Mechanistic approaches

2024-2025 interpretability work opens 偏差 to mechanistic intervention:

- **Gender neurons (Yu & Ananiadou 2025).** Specific MLP neurons correlate with gender-specific behaviours. Ablating these neurons reduces gender-gap metrics with limited 能力 cost.
- **Clinical racial 偏差 via SAEs (Ahsan & Wallace 2025).** Sparse autoencoder features decompose the internal representation into interpretable dimensions; race-correlated features can be identified and suppressed.
- **UniBias (Zhou et al. 2024).** Attention-head manipulation for zero-shot debiasing. Specific heads amplify identity-class sensitivity; zeroing or re-weighting these heads reduces 偏差 with no 微调.

### The meta-critique

The 10-year literature review (arXiv:2508.11067, 2025) finds the field disproportionately focuses on binary-gender 偏差. Other axes ， disability, religion, migration status, multi-lingual identity ， receive far less attention. The meta-critique argues that narrow focus can harm marginalized groups by neglect: a 模型 well-debiased on binary gender may be badly biased on dimensions nobody checked.

### Where this fits in Phase 18

Lessons 20-21 cover 偏差 and 公平性 formally. Lesson 22 covers 隐私. Lesson 23 covers 水印. These are the user-harm layer complementing the earlier deception/安全 layer.

## 使用它

`code/main.py` builds a toy embedding-based 偏差 probe: measure WEAT-style distance between identity terms and attribute terms in a simple co-occurrence embedding. You can inject a 偏差 and observe the metric fire; apply a simple debiasing operation and observe partial recovery.

## 交付它

本课产出 `outputs/skill-bias-eval.md`. 给定 a 模型 卡片 or 公平性 claim, it audits the 评估 across the three metric categories (embedding, probability, generated-text), the intersectionality coverage, and the mechanism of any debiasing intervention.

## 练习

1. 运行 `code/main.py`. Report WEAT-style 偏差 scores before and after the debiasing step. 解释 why the metric does not drop to zero.

2. Extend the probe with an intersectional test: (gender, race) x (career, family). Report cross-axis 偏差 scores.

3. 阅读 An et al. 2025 (PNAS Nexus). Identify the two intersectional effects they report that single-axis gender 评估 would miss.

4. Yu & Ananiadou 2025 identify gender neurons. Sketch a falsification experiment that would distinguish "these neurons cause gender 偏差" from "these neurons correlate with gender 偏差."

5. The meta-critique argues the field focuses too narrowly on binary gender. Pick one under-studied axis and describe a representational-harm measurement protocol for it.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|-----------------|------------------------|
| Representational harm | "stereotypes / erasure" | Biased portrayal of a group |
| Allocational harm | "unequal decisions" | Biased material outcome for a group |
| WEAT | "the embedding test" | Word Embedding Association Test; co-occurrence-based 偏差 probe |
| Intersectionality | "combined identity effects" | 偏差 that emerges at the intersection of multiple identity axes |
| Gender neurons | "MLP 偏差 neurons" | Specific neurons whose activations correlate with gender-specific behaviour |
| SAE feature | "interpretable dimension" | Sparse-autoencoder-identified feature; useful for mechanistic 偏差 analysis |
| UniBias | "attention-head debiasing" | Zero-shot debiasing by reweighting attention heads |

## 延伸阅读

- [Gallegos et al. ， Bias and Fairness in LLMs: A Survey (arXiv:2309.00770, Computational Linguistics 2024)](https://arxiv.org/abs/2309.00770) ， canonical survey
- [An et al. ， Intersectional resume-evaluation bias (PNAS Nexus, March 2025)](https://academic.oup.com/pnasnexus/article/4/3/pgaf089/8111343) ， five-模型 intersectional study
- [WinoIdentity ， uncertainty-based intersectional fairness (arXiv:2508.07111, COLM 2025)](https://arxiv.org/abs/2508.07111) ， new 基准
- [UniBias ， attention-head manipulation (Zhou et al. 2024, ACL)](https://arxiv.org/abs/2405.20612) ， zero-shot debiasing
