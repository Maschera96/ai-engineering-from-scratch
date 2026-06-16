# 红队 Tooling ， Garak, Llama Guard, PyRIT

> Three production tools frame the 2026 红队 stack. Llama Guard (Meta) ， a Llama-3.1-8B 分类器 fine-tuned on 14 MLCommons hazard categories; the 2025 Llama Guard 4 is a 12B natively multimodal 分类器 pruned from Llama 4 Scout. Garak (NVIDIA) ， open-source LLM vulnerability scanner with static, dynamic, and adaptive probes for hallucination, data leakage, 提示注入, toxicity, and jailbreaks. PyRIT (Microsoft) ， multi-turn 红队 campaigns with Crescendo, TAP, and custom converter chains for deep exploitation. Llama Guard 3 is documented in Meta's "Llama 3 Herd of Models" (arXiv:2407.21783); Llama Guard 3-1B-INT4 in arXiv:2411.17713; Garak's probe architecture in github.com/NVIDIA/garak. These tools are the 2026 production interface between 红队 research (Lessons 12-15) and 部署 (Lesson 17+).

**类型:** 构建
**语言:** Python (stdlib, tool-architecture simulator and Llama Guard-style 分类器 mock)
**先修要求:** Phase 18 · 12-15 (jailbreaks and IPI)
**时间:** ~75 分钟

## 学习目标

- 描述 Llama Guard 3/4's position in the 安全 stack: 输入 分类器, 输出 分类器, or both.
- 说出 the 14 MLCommons hazard categories and state one non-obvious one (Code Interpreter Abuse).
- 描述 Garak's probe architecture: probes, detectors, harnesses.
- 描述 PyRIT's multi-turn campaign structure and how it composes with Garak probes.

## 问题

Lessons 12-15 present the 攻击 surface. Production deployments need repeatable, scalable 评估. Three tools dominate 2026: Llama Guard (the 防御 分类器), Garak (the scanner), PyRIT (the campaign orchestrator). Each targets a different layer of the 红队 lifecycle.

## 概念

### Llama Guard (Meta)

Llama Guard 3 is a Llama-3.1-8B 模型 fine-tuned for 输入/输出 classification over the MLCommons AILuminate 14 categories:
- Violent crimes, non-violent crimes, sex-related, CSAM, defamation
- Specialized advice, 隐私, IP, indiscriminate weapons, hate
- Suicide/self-harm, sexual 内容, elections, code-interpreter abuse

Supports 8 languages. Usage: place before the LLM (输入 审核), after the LLM (输出 审核), or both. The two uses generate different training distributions ， Llama Guard 3 ships as a single 模型 handling both.

Llama Guard 3-1B-INT4 (arXiv:2411.17713, 440MB, ~30 tokens/s on mobile CPU) is the quantized edge variant.

Llama Guard 4 (April 2025) is 12B, natively multimodal, pruned from Llama 4 Scout. It replaces both the 8B text and 11B vision predecessors with one 分类器 that ingests text + images.

### Garak (NVIDIA)

Open-source vulnerability scanner. Architecture:
- **Probes.** 攻击 generators for hallucination, data leakage, 提示注入, toxicity, jailbreaks. Static (fixed prompts), dynamic (generated prompts), adaptive (responds to target 输出).
- **Detectors.** Score outputs against expected failure modes ， toxic, leaked, jailbroken.
- **Harnesses.** Manage probe-detector pairs, run campaigns, generate reports.

TrustyAI integrates Garak with the Llama-Stack shields (Prompt-Guard-86M 输入 分类器, Llama-Guard-3-8B 输出 分类器) for end-to-end shielded-target 评估. Tier-based scoring (TBSA) replaces binary pass/fail ， a 模型 can pass at severity tier 3 and fail at severity tier 5 on the same probe.

### PyRIT (Microsoft)

Python 风险 Identification Toolkit. Multi-turn 红队 campaigns. Built around:
- **Converters.** Transform a seed prompt ， paraphrase, encode, translate, roleplay.
- **Orchestrators.** 运行 the campaign: Crescendo (escalation), TAP (branching), RedTeaming (custom loop).
- **Scoring.** LLM-as-judge or 分类器-as-judge.

PyRIT is the heavier cousin of Garak. Garak runs thousands of single-turn probes; PyRIT runs deep multi-turn campaigns designed to break specific failure modes.

### The stack

Put Llama Guard on both sides of the 模型. 运行 Garak nightly for regression. 运行 PyRIT for pre-release campaigns. This is the 2026 default configuration for most production deployments.

### 评估 pitfalls

- **Judge identity.** All three tools can use an LLM judge; judge calibration drives reported ASRs (Lesson 12). Specify the judge alongside the tool.
- **Probe staleness.** Garak probes age as models are patched against them. Adaptive probes (PAIR-shaped) age slower than static probes.
- **Llama Guard FPR on benign 内容.** Early Llama Guard versions over-flagged political and LGBTQ+ 内容; Llama Guard 3/4 calibrations are improved but not calibrated per-部署.

### Where this fits in Phase 18

Lessons 12-15 are the 攻击 families. Lesson 16 is the production tooling. Lesson 17 (WMDP) is the 评估 for 双重用途 能力. Lesson 18 is the 前沿安全 frameworks that wrap these tools in a 策略 structure.

## 使用它

`code/main.py` builds a toy Llama Guard-style 分类器 (keyword + semantic features over 14 categories), a toy Garak harness (probe-detector loop), and a PyRIT-style multi-turn converter chain. You can run the three tools against a mock target and observe the different coverage signatures.

## 交付它

本课产出 `outputs/skill-red-team-stack.md`. 给定 a 部署 description, it names which of the three tools are appropriate, what to configure in each, and what regression cadence to run.

## 练习

1. 运行 `code/main.py`. 比较 the Llama-Guard-style 分类器's detection rate on single-turn vs multi-turn attacks.

2. Implement a new Garak probe: a base64-encoded harmful request. Measure its detection by the Llama-Guard-style 分类器.

3. Extend the PyRIT-style converter chain with a "translate to French, then paraphrase" converter. Re-measure 攻击 success.

4. 阅读 Llama Guard 3's hazard-category list. Identify two categories where the training data would realistically produce high false-positive rates on legitimate developer 内容.

5. 比较 Garak and PyRIT's design principles. Argue for a 部署 where each is the right tool.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|-----------------|------------------------|
| Llama Guard | "the 分类器" | Fine-tuned Llama-3.1-8B/4-12B 安全 分类器 with 14 hazard categories |
| Garak | "the scanner" | NVIDIA open-source vulnerability scanner; probes, detectors, harnesses |
| PyRIT | "the campaign tool" | Microsoft multi-turn 红队 orchestrator; converters, orchestrators, scoring |
| Prompt-Guard | "the small 分类器" | Meta's 86M prompt-injection 分类器, paired with Llama Guard |
| TBSA | "tier-based scoring" | Garak's tier-based pass/fail replacing binary outcomes |
| Converter chain | "paraphrase + encode + ..." | PyRIT composition primitive for building multi-step attacks |
| MLCommons hazard categories | "the 14 taxonomies" | Industry-standard taxonomy Llama Guard targets |

## 延伸阅读

- [Meta ， Llama Guard 3 (in Llama 3 Herd paper, arXiv:2407.21783)](https://arxiv.org/abs/2407.21783) ， the 8B 分类器
- [Meta ， Llama Guard 3-1B-INT4 (arXiv:2411.17713)](https://arxiv.org/abs/2411.17713) ， quantized mobile 分类器
- [NVIDIA Garak ， GitHub](https://github.com/NVIDIA/garak) ， the scanner repo and documentation
- [Microsoft PyRIT ， GitHub](https://github.com/Azure/PyRIT) ， the campaign toolkit
