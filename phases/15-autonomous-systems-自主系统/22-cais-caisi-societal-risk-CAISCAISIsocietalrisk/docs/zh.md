# CAIS、CAISI 与社会规模风险

> The Center for AI 安全 (CAIS, San Francisco, founded 2022 by Hendrycks and Zhang) publishes the four-风险 framework — malicious use, AI races, organizational 风险, rogue AIs — and the May 2023 statement on extinction 风险 signed by hundreds of professors and company leaders. 2026 releases 来自 CAIS: AI Dashboard for frontier-模型 评估, Remote Labor Index (带有 Scale AI), Superintelligence Strategy Paper, AI Frontiers newsletter. A distinct entity: NIST Center for AI Standards and Innovation (CAISI) — US-government-facing voluntary agreements and unclassified 能力 评估s focused on cyber, bio, and chemical-weapons 风险. CAIS flags organizational 风险 as one of four top-level 风险: 安全 culture, rigorous audits, multi-layered 防御, and information security are foundational but routinely traded off 针对 部署 speed. California SB-53, if signed, would be the first US 说明-level catastrophic-风险 regulation.

**Type:** Learn
**Languages:** Python (stdlib, four-风险 inventory and 缓解措施 matcher)
**Prerequisites:** Phase 15 · 19 (RSP), Phase 15 · 20 (PF + FSF)
**Time:** ~45 minutes

## 问题

Lessons 19 and 20 covered lab-内部 scaling 政策. Lesson 21 covered 独立 能力 评估. This lesson covers the third perspective: civil society and government organizations 人员 shape public discussion and regulatory baseline for catastrophic AI 风险.

Two distinct entities matter. CAIS is a non-profit 研究 org that publishes frameworks for thinking about AI 风险 and coordinates public statements. CAISI is a US-government center within NIST that runs voluntary agreements 带有 labs and unclassified 能力 评估s. The names rhyme; the missions do 不overlap. A practitioner 应该 know 两者.

这个practical content: CAIS's four-风险 framework is the most widely cited societal-scale-风险 分类法 in the literature. 安全 culture and organizational 风险 are one of those four, and this is the one most 直接 under a practitioner's 控制. SB-53 (California) would be the first US 说明-level catastrophic-风险 regulation if signed; the bill's framing matters because 说明-level regulation has historically led federal action in US tech 政策.

## 概念

### CAIS — Center for AI 安全

- Founded: 2022 in San Francisco, by Dan Hendrycks and colleagues (the "Zhang" 命名 refers to an early collaborator, 不a current co-founder; see CAIS website for current leadership).
- Status: 501(c)(3) non-profit.
- Notable 2023 输出: statement on extinction 风险, co-signed by hundreds of researchers and CEOs. Stated: "Mitigating the 风险 of extinction 来自 AI 应该 be a global priority alongside other societal-scale 风险 such as pandemics and nuclear war."
- 2026 输出: AI Dashboard for frontier-模型 评估, Remote Labor Index (joint 带有 Scale AI), Superintelligence Strategy Paper, AI Frontiers newsletter.

### 这个four-风险 framework

CAIS's framework groups catastrophic AI 风险 进入 four top-level 类别:

1. **Malicious use**: a bad actor uses AI to cause harm (bioweapons synthesis, disinformation, cyberattacks).
2. **AI races**: competitive pressure between labs, companies, or nations pushes 部署 past the point 在哪里 it is safe.
3. **Organizational 风险**: 内部 lab dynamics (安全-culture 失败, insufficient 审计, under-resourced security) 生成 a bad 部署.
4. **Rogue AIs**: a sufficiently capable AI pursues goals that conflict 带有 人类 welfare.

这is 不the 只 分类法; it is the most cited. The 类别 are 不mutually exclusive — a rogue AI produced by an organization that traded 审计 for speed in a race is 所有 four.

### 位置 organizational 风险 lives

Of the four 类别, organizational 风险 is the most actionable for practitioners. A lab's 安全 culture, 审计 rigor, 防御 layering, and information security decide 是否 their 模型 ships 带有 the 控制 of Lessons 10–18 actually in place, or 是否 those 控制 are checklist items nobody verified.

这个concrete organizational-风险 levers:

- **安全 culture**: do team members feel able to escalate a concern 没有 career 成本? CAIS surveys find this is a strong predictor of the other levers.
- **Rigorous audits**: 外部 and 内部. 内部-只 audits 生成 optimistic reports.
- **Multi-layered 防御**: no 单一 层 is sufficient (the running theme of Phase 15).
- **Information security**: 模型 权重 leaking, 评估 data leaking, monitor-bypass techniques leaking. RAND SL-4 in Lesson 19 is a 具体 standard.

### CAISI — Center for AI Standards and Innovation

- Operates within NIST.
- Runs voluntary agreements 带有 frontier labs.
- Publishes unclassified 能力 评估s focused on cyber, bio, and chemical-weapons 风险.
- Distinct 来自 CAIS; the acronyms collide; check the URL (nist.gov) to 确认 哪个 one you are reading.

CAISI's role is the public, government-facing counterpart to METR's private lab engagements (Lesson 21). CAISI reports are unclassified; METR reports are often NDA-gated. A practitioner reading 两者 gets a fuller picture.

### California SB-53

这个California Senate bill (2025–2026 session) addresses catastrophic 风险 来自 前沿模型. Key provisions as drafted:

- 具体 能力 阈值 that 触发器 说明-level obligations.
- Whistleblower protections for AI lab employees.
- Incident reporting requirements for catastrophic 失败.

如果signed, it would be the first US 说明-level catastrophic-风险 regulation. Regardless of signing status, the bill's framing shapes 如何 other 说明 legislatures approach the problem. Practitioners in California 应该 跟踪 the bill's status; practitioners elsewhere 应该 阅读 it to understand 什么 US 说明-level regulation will likely look like.

### Societal-scale 风险 is 不a 单一-层 problem

这个running theme of Phase 15 — 防御 in depth — applies at the societal 层 too. No 单一 organization, regulation, or framework closes catastrophic 风险. The ecosystem functions 只 when:

- Labs ship scaling 政策 (Lessons 19, 20).
- 外部 评估器s 生成 measurements (Lesson 21).
- Civil society tracks and publicizes (CAIS).
- Government runs voluntary programs and baseline regulation (CAISI, SB-53).
- Practitioners build multi-layered 控制 (Lessons 10–18).

这is the final synthesis for the phase: 每个 previous lesson is one 层 in a stack whose completeness matters more than 任何 单一 层's strength.

## 使用它

`code/main.py` implements a small 风险-inventory 工具. 给定a 拟议的 部署, it tags the 部署 针对 the four-风险 类别 and returns a 缓解措施 checklist. It's a reading aid for the framework, 不a substitute for 人类 judgment.

## 交付它

`outputs/skill-societal-risk-review.md` reviews a 部署 for societal-scale-风险 posture: 哪个 of the four 类别 it touches, 什么 缓解措施 are in place, 什么 the organizational-风险 exposure is.

## 练习

1. 运行 `code/main.py`. Feed in three synthetic 部署s at different scales. 确认 the four-风险 tags match 什么 you would expect; 识别 one case 在哪里 the 工具 under- or over-tags.

2. 阅读 the CAIS four-风险 论文 in full. Pick one 风险 类别 and 编写 two paragraphs on 什么 you believe is the most important 2026 development in that 类别.

3. 阅读 a current draft of California SB-53. 识别 one provision you believe strengthens the catastrophic-风险 posture and one you believe weakens it. Justify 两者.

4. Pick a 生产环境 AI 部署 you know (yours or a published one). 分数 it 针对 the organizational-风险 sub-levers: 安全 culture, 审计 rigor, multi-layered 防御, information security. Which is weakest? 内容 would it 成本 to bring it to par?

5. Sketch a 2028 版本 of the four-风险 framework that reflects one year of additional 能力 and one year of additional 部署 experience. 内容 would you add, remove, or regroup?

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| CAIS | "Center for AI 安全" | Non-profit; four-风险 framework; 2023 extinction statement |
| CAISI | "US government AI 安全" | NIST Center; voluntary agreements; unclassified evals |
| Four-风险 framework | "CAIS's 分类法" | malicious use, AI races, organizational 风险, rogue AIs |
| Malicious use | "Bad actor uses AI" | Bioweapons, disinformation, cyberattacks |
| AI races | "Competitive pressure" | Labs/companies/nations push 部署 past 安全 |
| Organizational 风险 | "Lab 内部 失败" | 安全 culture, 审计, 防御, infosec |
| Rogue AI | "Misaligned 智能体" | Capable AI pursuing goals conflicting 带有 人类 welfare |
| California SB-53 | "说明-level regulation" | 2025–2026 bill; first US 说明 catastrophic-风险 regulation if signed |

## 延伸阅读

- [Center for AI Safety](https://safe.ai/) — institutional home of the four-风险 framework.
- [CAIS — AI Risks that Could Lead to Catastrophe](https://safe.ai/ai-risk) — the four-风险 论文.
- [CAIS — May 2023 statement on extinction risk](https://safe.ai/statement-on-ai-risk) — short joint statement.
- [NIST CAISI](https://www.nist.gov/caisi) — government-facing AI standards and innovation center.
- [Anthropic — Measuring agent autonomy in practice](https://www.anthropic.com/research/measuring-agent-autonomy) — connects lab-level commitments to societal-scale framing.
