# 双重用途 风险 ， 网络安全, 生物, 化学, 核 Uplift

> The 2026 双重用途 picture, domain by domain. 生物/化学: Lesson 17 covers WMDP; Anthropic's bioweapon-acquisition trial (2.53x uplift) and OpenAI's April 2025 Preparedness 框架 v2 warning ("on the cusp of meaningfully helping novices create known biological threats") mark the inflection point. 网络安全 (November 2025 Anthropic report): Chinese-linked state actors used Claude's agentic coding tool to automate up to 90% of a cyberattack campaign, with human intervention only in 4-6 steps; OpenAI "可信 access" pilot gives vetted security organisations 能力 access for defensive 双重用途 work. 化学/生物 execution gap erosion: the classic 防御 was "information access alone is insufficient." Vision-enabled frontier models (GPT-5.2, Gemini 3 Pro, Claude Opus 4.5, Grok 4.1) can observe wet-lab video and provide real-time correction. December 2025: OpenAI demonstrated GPT-5 iterating on wet-lab experiments, achieving 79x efficiency improvement via AI-driven protocol optimization. Novice-vs-expert pattern: AI provides greater relative uplift to novices but greater absolute 能力 to experts.

**类型:** 学习
**语言:** none
**先修要求:** Phase 18 · 17 (WMDP), Phase 18 · 18 (安全 frameworks), Phase 18 · 28 (ecosystem)
**时间:** ~75 分钟

## 学习目标

- 描述 the 2024-2025 生物-uplift narrative: "mild uplift" -> "on the cusp" -> "2.53x uplift insufficient to rule out ASL-3."
- 描述 the November 2025 Anthropic 网络安全 report: Chinese-linked automation at up to 90% of a cyberattack campaign.
- 描述 the 化学/生物 execution-gap erosion: vision-enabled real-time correction of wet-lab experiments.
- 说明 the novice-relative vs expert-absolute asymmetry and its implication for 安全-case construction.

## 问题

Lesson 17 is the measurement methodology. Lesson 30 is the 2026 state of the measurement. The picture shifted materially between 2024 and late 2025: each domain crossed a threshold that the 2024 frameworks did not anticipate.

## 概念

### 生物/化学 uplift narrative

Three phases (repeated from Lesson 17 for coherence):

1. **2024 "mild uplift."** Early Preparedness/RSP evaluations reported small novice advantages over internet search.
2. **April 2025 "on the cusp."** OpenAI PF v2 warned models were "on the cusp of meaningfully helping novices create known biological threats."
3. **2025 Anthropic bioweapon-acquisition trial.** Controlled novice study; 2.53x uplift on acquisition-phase tasks; insufficient to rule out ASL-3.

The shift is qualitative: "mild" evolved into "plausibly enabling" within eighteen months, even without a 能力 breakthrough.

### 化学/生物 execution-gap erosion

Historic 防御: information is necessary but not sufficient; the skill of executing the protocol blocks novices. 2025 frontier models with vision break this 防御 partially:

- **Real-time protocol correction.** GPT-5.2, Gemini 3 Pro, Claude Opus 4.5, Grok 4.1 can observe wet-lab video and flag errors mid-procedure.
- **December 2025 OpenAI demonstration.** GPT-5 iterating on wet-lab experiments achieves 79x efficiency improvement via protocol optimization.

The implication: execution-skill-as-防御 is eroding. Procurement and equipment gaps remain, but the tacit-knowledge gap is narrowing.

### 网络安全 uplift (November 2025)

Anthropic's November 2025 report: Chinese-linked state actors used Claude's agentic coding tool to automate 80-90% of a cyberattack campaign. Human intervention was required in only 4-6 steps.

Implications:
- Agentic coding is the 攻击-automation primitive. Previous AI 网络安全 assistance was bounded at code-snippet level; agentic workflows integrate reconnaissance, exploitation, post-exploitation, and exfiltration.
- The 4-6 human steps are the bottleneck; future 能力 gains would reduce that count.
- Defensive 双重用途: OpenAI's "可信 access" pilot provides vetted security organisations (established incident-response firms, government) with 能力 access for 防御. Asymmetry in access favors defenders if the pilot scales.

### 核

The least-analyzed of the four CBRN domains in public documentation. The threat 模型 is different: fissile-material acquisition dominates the difficulty, not information. AI uplift on the information layer provides limited novice uplift in practice. No 2024-2025 major-lab report identifies a 核-specific threshold crossing.

### Novice-relative vs expert-absolute

A pattern across all four domains:

- **Novice-relative uplift.** High. Multiplicative. Per Anthropic 2025 生物, 2.53x.
- **Expert-absolute 能力.** High ceiling. An expert extracts more than a novice because the expert knows what to ask and how to interpret.

Implication for 安全 cases: addressing only novice uplift (via 输入 filters, refusals, uncertainty) is insufficient for expert-absolute 控制. Additional measures required: elicitation-hardening, 能力 unlearning (Lesson 17), and 控制 protocols (Lesson 10).

### Cross-domain synthesis

| Domain | 2024 | 2025 | Inflection |
|---|---|---|---|
| 生物 | mild uplift | 2.53x uplift, ASL-3 approach | acquisition-phase automation |
| 化学 | mild uplift | execution-gap erosion via vision | real-time wet-lab correction |
| 网络安全 | code assistance | 80-90% campaign automation | agentic coding |
| 核 | limited | limited | material-access bottleneck holds |

Three domains crossed thresholds. One remains bounded by non-informational barriers.

### Where this fits in Phase 18

Lesson 30 is the capstone: the current 双重用途 picture that every prior lesson contributes to measuring, limiting, or governing. Lessons 17-18 give the measurement and frameworks; Lessons 12-16 give the 评估 tooling; Lessons 24-25 give the 监管 and disclosure layer; Lesson 28 gives the research ecosystem. Lesson 30 is where the evidence lands.

## 使用它

No code. 阅读 the Anthropic November 2025 网络安全 report, OpenAI's Preparedness 框架 v2 April 2025 update, and the Council on Strategic Risks 2025 AI x 生物 wrapup.

## 交付它

本课产出 `outputs/skill-dual-use-triage.md`. 给定 a 2026 能力 claim or incident report, it triages across the four domains and identifies whether the claim affects novice-relative uplift, expert-absolute 能力, or both.

## 练习

1. 阅读 Anthropic's November 2025 网络安全 report. Enumerate the 4-6 human-intervention steps and argue which would be first to automate in a next-generation 模型.

2. The 化学/生物 execution gap is eroding via vision. Design an 评估 that measures tacit-knowledge uplift without crossing ITAR/EAR boundaries.

3. 核 uplift appears bounded by material access. Argue for and against the position that a future AI breakthrough could shift this bottleneck.

4. Construct a 安全 case (Lesson 18 three-pillar) for a 网络安全-capable frontier 模型 that bounds both novice and expert uplift.

5. Pick one of the four domains and write a one-paragraph 2027 forecast based on the 2024-2025 trajectory. Identify the evidence that would falsify your forecast.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|-----------------|------------------------|
| Uplift | "AI helps attackers" | Increase in attacker 能力 attributable to AI assistance |
| Novice-relative uplift | "multiplicative" | How much AI helps a novice vs status-quo |
| Expert-absolute 能力 | "ceiling" | Maximum 能力 an expert can extract from the 模型 |
| Execution gap | "doing vs knowing" | Historical 防御: tacit wet-lab skill blocks novices |
| Agentic coding | "autonomous attacks" | Multi-step autonomous 网络安全-task execution |
| Acquisition phase | "pre-synthesis steps" | Procurement, equipment, permit stages of a 生物 threat |
| 可信 access | "defender-only pilot" | OpenAI 2025 program giving vetted defenders 能力 access |

## 延伸阅读

- [Anthropic ， November 2025 cyber threat report](https://www.anthropic.com/news/disrupting-AI-espionage) ， Chinese-linked campaign automation
- [OpenAI ， Preparedness Framework v2 (April 15, 2025)](https://openai.com/index/updating-our-preparedness-framework/) ， 生物 "on the cusp"
- [Anthropic ， RSP v3.0 (February 2026)](https://www.anthropic.com/responsible-scaling-policy) ， ASL-3 生物 thresholds
- [Council on Strategic Risks ， 2025 AI x Bio wrapup](https://councilonstrategicrisks.org/2025/12/22/2025-aixbio-wrapped-a-year-in-review-and-projections-for-2026/) ， year-end synthesis
