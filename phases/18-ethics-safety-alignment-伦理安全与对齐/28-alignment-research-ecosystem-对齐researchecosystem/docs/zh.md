# 对齐 Research Ecosystem ， MATS, Redwood, Apollo, METR

> Five organisations define the 2026 non-lab 对齐 research layer. MATS (ML 对齐 & Theory Scholars): 527+ researchers since late 2021, 180+ papers, 10K+ citations, h-index 47; summer 2024 cohort incorporated as 501(c)(3) with ~90 scholars and 40 mentors; 80% of pre-2025 alumni work on 安全/security with 200+ at Anthropic, DeepMind, OpenAI, UK AISI, RAND, Redwood, METR, Apollo. Redwood Research: applied 对齐 lab founded by Buck Shlegeris; introduced AI 控制 (Lesson 10); collaborates with UK AISI on 控制 安全 cases. Apollo Research: pre-部署 密谋 evaluations for frontier labs; authored In-Context 密谋 (Lesson 8) and Towards 安全 Cases for AI 密谋. METR (模型 评估 and Threat Research): task-based 能力 evaluations, autonomous-task time-horizon studies; "Common Elements of Frontier AI 安全 Policies" compares lab frameworks. Eleos AI Research: 模型-welfare pre-部署 evaluations (Lesson 19); conducted Claude Opus 4 welfare assessment.

**类型:** 学习
**语言:** none
**先修要求:** Phase 18 · 01-27 (prior Phase 18 lessons)
**时间:** ~45 分钟

## 学习目标

- Identify the five organisations of the non-lab 对齐 research ecosystem and their core 输出.
- 描述 MATS's scale (scholars, papers, h-index) and its role as a talent pipeline.
- 描述 Redwood's AI 控制 agenda and its partnership with UK AISI.
- 描述 METR's task-based 评估 methodology.

## 问题

The frontier labs (Lesson 18) produce 安全 evaluations internally and publish selected results. The ecosystem outside the labs is where the evaluations are validated, where novel failure modes are first discovered, and where talent is trained. Understanding the ecosystem helps interpret which research findings are 可信 by whom.

## 概念

### MATS (ML 对齐 & Theory Scholars)

Started late 2021. Research mentorship program; scholars spend 10-12 weeks with a senior researcher on a specific 对齐 problem.

Scale (2026):
- 527+ researchers since inception.
- 180+ papers published.
- 10K+ citations.
- h-index 47.
- Summer 2024: 90 scholars + 40 mentors; incorporated as 501(c)(3).

Career outcomes: ~80% of pre-2025 alumni are working on 安全/security. 200+ at Anthropic, DeepMind, OpenAI, UK AISI, RAND, Redwood, METR, Apollo.

### Redwood Research

Applied 对齐 lab. Founded by Buck Shlegeris. Introduced the AI 控制 agenda (Lesson 10). Collaborates with UK AISI on 控制 安全 cases. Advises DeepMind and Anthropic on 评估 design.

Canonical papers: Greenblatt, Shlegeris et al., "AI 控制" (arXiv:2312.06942, ICML 2024); 对齐 Faking (Greenblatt, Denison, Wright et al., arXiv:2412.14093, joint with Anthropic).

Style: specific threat models, worst-case adversaries, concrete protocols that can be stress-tested.

### Apollo Research

Pre-部署 密谋 evaluations for frontier labs. Authored In-Context 密谋 (Lesson 8, arXiv:2412.04984). Partner on 2025 OpenAI anti-密谋 training collaboration. Produces Towards 安全 Cases for AI 密谋 (2024).

Style: agentic-setting evaluations where deception can emerge; three-pillar decomposition (misalignment, goal-directedness, situational awareness).

### METR (模型 评估 and Threat Research)

Task-based 能力 evaluations. Autonomous-task completion time-horizon studies. "Common Elements of Frontier AI 安全 Policies" (metr.org/common-elements, 2025) compares lab frameworks.

Co-author on AI 密谋 安全-case sketch with Apollo.

Style: long-horizon task evaluations, empirical 能力 measurement, 框架 synthesis.

### Eleos AI Research

模型-welfare pre-部署 evaluations. Conducted the Claude Opus 4 welfare assessment documented in section 5.3 of the 系统 卡片. Provides the external methodology check for Lesson 19's welfare-relevant claims.

### The flow

MATS trains researchers. Graduates go to Anthropic, DeepMind, OpenAI (lab 安全 teams) or to Redwood, Apollo, METR, Eleos (external 评估). External evaluators partner with labs and with UK AISI / CAISI. Publications feed the ecosystem back to MATS for the next cohort.

### Why this layer matters

Single-source evaluations are unreliable: labs evaluating their own models have a structural conflict of interest. External evaluators can raise and validate failure modes the lab may underreport. The 2024 休眠智能体 paper (Lesson 7) was Anthropic + Redwood; 对齐 Faking was Anthropic + Redwood; In-Context 密谋 was Apollo; Anti-密谋 was Apollo + OpenAI. The multi-org structure is the quality 控制.

### Where this fits in Phase 18

Lessons 7-11 reference Redwood and Apollo work; Lesson 18 references METR's 框架 comparison; Lesson 19 references Eleos. Lesson 28 is the explicit organisational map for the ecosystem the rest of the Phase relies on.

## 使用它

No code. 阅读 METR's "Common Elements of Frontier AI 安全 Policies" as an example of how external synthesis adds value to lab-internal 策略 work.

## 交付它

本课产出 `outputs/skill-ecosystem-map.md`. 给定 an 对齐 claim or 评估, it identifies the organisation, the publication venue, and the methodological style, and cross-checks against known-counterpart organisations.

## 练习

1. Pick one paper from Lessons 7-15 and identify the organisations involved. Cross-check the authors against MATS alumni and current ecosystem affiliations.

2. 阅读 METR's "Common Elements of Frontier AI 安全 Policies." Identify the three cross-lab convergences they emphasize and the two largest divergences.

3. MATS career outcomes are ~80% 安全/security. Argue whether this selection pressure is adaptive (trains the field) or biased (filters out heterodox positions).

4. Redwood and Apollo both do 控制/密谋 work but with different styles. Pick a failure mode and describe how each would investigate it.

5. Eleos AI is the only pure 模型-welfare organisation. Design a hypothetical second organisation focused on a different welfare-adjacent question (cognitive liberty, robotic embodiment, etc.) and articulate its methodology.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|-----------------|------------------------|
| MATS | "the mentorship program" | ML 对齐 & Theory Scholars; 527+ researchers since 2021 |
| Redwood Research | "the 控制 lab" | Applied 对齐; AI 控制 authors; UK AISI partner |
| Apollo Research | "the 密谋 evals" | Pre-部署 密谋 evaluations for frontier labs |
| METR | "the task-horizon evals" | Task-based 能力 evaluations; 框架 synthesis |
| Eleos AI | "the welfare lab" | 模型-welfare pre-部署 evaluations |
| Talent pipeline | "MATS -> labs" | MATS graduates flow to Anthropic, DM, OpenAI, Redwood, Apollo, METR |
| External 评估 | "non-lab check" | 评估 not done by the 模型's producer; adds credibility |

## 延伸阅读

- [MATS (ML Alignment & Theory Scholars)](https://www.matsprogram.org/) ， the mentorship program
- [Redwood Research](https://www.redwoodresearch.org/) ， AI 控制 papers
- [Apollo Research](https://www.apolloresearch.ai/) ， 密谋 evaluations
- [METR ， Common Elements of Frontier AI Safety Policies](https://metr.org/blog/2025-03-26-common-elements-of-frontier-ai-safety-policies/) ， 框架 comparison
- [Eleos AI Research](https://www.eleosai.org/research) ， 模型福利 methodology
