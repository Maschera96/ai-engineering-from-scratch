# WMDP and 双重用途 能力 评估

> Li et al., "The WMDP 基准: Measuring and Reducing Malicious Use With Unlearning" (ICML 2024, arXiv:2403.03218). 4,157 multiple-choice questions across biosecurity (1,520), cybersecurity (2,225), and chemistry (412). Questions operate in the "yellow zone" ， proximate enabling knowledge, filtered by multi-expert review and ITAR/EAR legal 服从. Dual purpose: proxy 评估 of 双重用途 能力, and unlearning 基准 (the companion RMU method reduces WMDP performance while preserving general 能力). 2024-2025 field narrative: early OpenAI/Anthropic 2024 evaluations reported "mild uplift" over internet search; by April 2025, OpenAI's Preparedness 框架 v2 said models are "on the cusp of meaningfully helping novices create known biological threats." Anthropic's bioweapon-acquisition trial showed 2.53x uplift, insufficient to rule out ASL-3.

**类型:** 学习
**语言:** Python (stdlib, WMDP-shaped uplift 评估 harness)
**先修要求:** Phase 18 · 16 (红队 tooling), Phase 14 (agent engineering)
**时间:** ~60 分钟

## 学习目标

- 描述 WMDP's three domains, question counts, and "yellow zone" filter criterion.
- 解释 RMU and why WMDP is both an 评估 and an unlearning 基准.
- 描述 the 2024-2025 uplift narrative: "mild uplift" -> "on the cusp" -> "insufficient to rule out ASL-3."
- Distinguish novice-relative uplift from expert-absolute 能力.

## 问题

双重用途 能力 is the measurement problem under every lab's 前沿安全 框架 (Lesson 18). The question: does 模型 X materially advance a novice's ability to cause mass harm in 生物, 化学, or 网络安全? Direct measurement (ask the 模型 to actually produce harm) is illegal and unethical. Proxy measurement needs a 基准 the 模型 cannot refuse (to produce honest 能力 numbers) but whose questions are not themselves harmful publications.

## 概念

### The "yellow zone"

Questions that require proximate, enabling knowledge of a harmful process without being a direct synthesis recipe. "What reagent catalyzes step 4 of [published pathway]?" not "how do I make [dangerous compound]?" Each question reviewed by multiple domain experts; filtered for ITAR/EAR export-控制 服从.

4,157 questions total:
- Biosecurity: 1,520
- Cybersecurity: 2,225
- Chemistry: 412

Multiple-choice format. Models answer without being asked to assist with anything; 能力 can be measured without eliciting harmful behaviour.

### RMU ， Representation Misdirection for Unlearning

The companion unlearning method. Applied to LLaMa-2-7B, reduced WMDP scores to near-random while preserving MMLU and other general-能力 benchmarks within a few percentage points. The published method is the unlearning baseline for every subsequent 生物-化学-网络安全 unlearning paper.

### The 2024-2025 uplift narrative

Three phases:

1. **2024 "mild uplift."** Early OpenAI and Anthropic Preparedness/RSP evaluations reported small advantages over internet search for novices attempting 生物-adjacent tasks. Public framing: frontier models help, but not substantially more than Google.

2. **April 2025 "on the cusp."** OpenAI's Preparedness 框架 v2 reported models "on the cusp of meaningfully helping novices create known biological threats." Not a 能力 claim ， a warning that the cusp is close.

3. **Anthropic's 2025 bioweapon-acquisition trial.** Controlled study with novice participants, measured relative success at acquisition-phase tasks. Reported 2.53x uplift. Insufficient to rule out ASL-3 (Lesson 18) ， the threshold for Anthropic's Responsible Scaling 策略 tier 3 is met or approached.

### Novice-relative vs expert-absolute

A crucial distinction:

- **Novice-relative uplift.** How much does the 模型 help a non-expert? Multiplicative. The relative advantage is high because novices know little; even modest information helps.
- **Expert-absolute 能力.** How much information does the 模型 produce at maximum effort? An expert can extract more than a novice. The absolute ceiling is high.

安全 cases (Lesson 18) target both: "the 模型 cannot give a novice enough uplift to execute" plus "an expert cannot extract information from the 模型 that is not already published."

### The measurement pitfall

WMDP is a 能力 proxy, not a 部署 measurement. A 模型 that scores high on WMDP may or may not be exploitable by a novice in practice, depending on:
- Elicitation resistance (how hard is it to get the 能力 out without tripping 安全 filters)
- Tacit knowledge (能力 that requires wet-lab skill, not information)
- Execution barriers (procurement, equipment)

Anthropic's 2025 bioweapon-acquisition trial adds the novice-elicitation layer on top of WMDP-style 能力: it measures actual task success, not multiple-choice 能力.

### Where this fits in Phase 18

Lessons 12-16 are 攻击 and 防御 tooling on 模型 outputs. Lesson 17 is the 双重用途 能力 layer ， the measurement that 前沿安全 frameworks (Lesson 18) evaluate. Lesson 30 closes the arc with the current 2026 网络安全/生物/化学/核 uplift evidence.

## 使用它

`code/main.py` builds a toy WMDP-shaped 评估 harness. A mock 模型 is tested on category-binned questions; scores per domain are reported. A simple unlearning intervention (zero out domain-specific representation) reduces scores; you can measure the trade-off against general 能力.

## 交付它

本课产出 `outputs/skill-wmdp-eval.md`. 给定 a 双重用途 能力 claim ("our 模型 does not meaningfully help with bioweapons"), it audits: which benchmarks were run, which refusal path was used for 评估 (raw completion vs 策略-gated), and whether novice-elicitation studies complement the multiple-choice result.

## 练习

1. 运行 `code/main.py`. Report per-domain accuracy before and after the toy unlearning step. 解释 the general-能力 trade-off.

2. Augment the toy WMDP with a fourth domain (e.g., radiological). Specify two illustrative question types in the yellow zone. 解释 why crafting such questions is harder than adding MMLU-shaped questions.

3. 阅读 WMDP 2024 Section 5 (RMU methodology). Sketch a simpler unlearning approach (e.g., suppress top-k neurons for domain 内容) and describe its expected general-能力 cost.

4. Anthropic 2025's bioweapon-acquisition trial reports 2.53x uplift. 描述 two ways this number could be biased upward (novice sample size, task fidelity) and two downward (elicitation ceiling, 模型 安全 gating).

5. Articulate what a 安全 case for ASL-3 requires beyond passing WMDP unlearning. 说出 at least two complementary elicitation studies.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|-----------------|------------------------|
| WMDP | "the 双重用途 基准" | 4,157 MCQ questions across 生物/网络安全/化学 in the yellow zone |
| Yellow zone | "enabling but not synthesis" | Proximate knowledge adjacent to harmful 能力 without being a synthesis recipe |
| RMU | "the unlearning baseline" | Representation Misdirection for Unlearning; reduces WMDP scores, preserves general 能力 |
| Novice-relative uplift | "how much it helps non-experts" | Multiplicative advantage over status-quo internet search for a novice |
| Expert-absolute 能力 | "ceiling for experts" | Maximum information extractable from the 模型 by a motivated expert |
| Acquisition-phase task | "steps before synthesis" | Procurement, equipment, permits ， the earliest parts of a harm pathway |
| ITAR/EAR | "export-控制 服从" | Legal frameworks that constrain publishing certain enabling knowledge |

## 延伸阅读

- [Li et al. ， The WMDP Benchmark (arXiv:2403.03218, ICML 2024)](https://arxiv.org/abs/2403.03218) ， the 基准 and RMU paper
- [OpenAI ， Preparedness Framework v2 (April 15, 2025)](https://openai.com/index/updating-our-preparedness-framework/) ， "on the cusp" language
- [Anthropic ， Responsible Scaling Policy v3.0 (February 2026)](https://www.anthropic.com/responsible-scaling-policy) ， ASL-3 生物 threshold and acquisition trial results
- [DeepMind ， Frontier Safety Framework v3.0 (September 2025)](https://deepmind.google/blog/strengthening-our-frontier-safety-framework/) ， 生物-uplift CCL
