# Anthropic Responsible Scaling 政策 v3.0

> RSP v3.0 went 进入 effect February 24, 2026, replacing the 2023 政策. Two-tier 缓解措施: 什么 Anthropic will do unilaterally vs 什么 is framed as an 行业范围 建议 (including RAND SL-4 security standards). Adds Frontier 安全 Roadmaps and 风险 Reports as standing documents rather than one-off deliverables. Drops the 2023 暂停 承诺. Introduces the AI R&D-4 阈值: once crossed, Anthropic 必须 publish an affirmative case identifying mis对齐 风险 and 缓解措施. Claude Opus 4.6 does 不cross it. Anthropic states in the v3.0 announcement that "confidently ruling this out is becoming difficult." SaferAI rated the 2023 RSP at 2.2; they downgraded v3.0 to 1.9, putting Anthropic in the "weak" RSP 类别 alongside OpenAI and DeepMind. 定性 阈值 replaced the 2023 定量 commitments; removing the 暂停 条款 is the sharpest 回归.

**Type:** Learn
**Languages:** Python (stdlib, RSP 阈值 decision engine)
**Prerequisites:** Phase 15 · 06 (AAR), Phase 15 · 07 (RSI)
**Time:** ~45 minutes

## 问题

Frontier labs publish scaling 政策 that are partly technical documents, partly governance documents, and partly signals to regulators. RSP v3.0 is the current Anthropic document. Reading it closely matters 不because compliance 带有 it is binding (it is not), but because the framing shapes 如何 a lab conceives of catastrophic 风险 and 如何 they communicate trade-offs to the public.

这个v3.0 vs v2.0 diff is the useful unit. 内容 got added: Frontier 安全 Roadmaps, 风险 Reports, the AI R&D-4 阈值. 内容 got removed: the 2023 暂停 承诺. 内容 got reframed: a two-tier 缓解措施 schedule split between Anthropic-单边 and industry-建议. 外部 审查 — SaferAI — downgraded the 评分 来自 2.2 (v2) to 1.9 (v3.0). This is 如何 a scaling 政策 can get less rigorous while looking more polished.

## 概念

### 这个two-tier 缓解措施 schedule

- **Anthropic 单边 actions**: 什么 Anthropic will do regardless of 什么 other labs do. Training stops above a 阈值, 具体 security measures, 具体 部署 gates.
- **行业范围 recommendations**: 什么 Anthropic thinks the industry 应该 do collectively. Includes RAND SL-4 security standards. These are 不commitments on Anthropic's side; they are 政策 advocacy.

这个two-tier structure was 不in v2. It means that a reader needs to look at 哪个 column each 承诺 lives in. A security measure in the "行业范围 建议" column is 不Anthropic's promise; it is Anthropic's hope.

### 这个AI R&D-4 阈值

这is the 能力 level RSP v3.0 names as the important next 阈值. Specifically: a 模型 that could automate a substantial fraction of AI 研究 at competitive 成本. Once Anthropic believes a 模型 crosses it, they 必须 publish an affirmative case identifying mis对齐 风险 and 缓解措施 之前 continued scaling.

Claude Opus 4.6 does 不cross it per the v3.0 announcement. The document adds: "confidently ruling this out is becoming difficult." That phrasing matters; it concedes that the 阈值 is close enough to be a live concern, 不a speculative 限制.

Lesson 6 (Automated 对齐 Research) and Lesson 7 (Recursive Self-Improvement) feed 直接 进入 this 阈值. Automated 对齐 researchers crossing 研究-质量 bars is 证据 that the AI R&D-4 阈值 is approaching.

### Frontier 安全 Roadmaps and 风险 Reports

v3.0 elevates two artifact types to standing documents:

- **Frontier 安全 路线图**: forward-looking document describing planned 安全 work, 能力 expectations, and 缓解措施 研究.
- **风险 报告**: retrospective document on 具体 models 之后 release, describing observed 能力 and residual 风险.

两者 are public. 两者 are updated on a declared 节奏. The utility is: reader can 跟踪 如何 什么 Anthropic said they would do in a 路线图 compares to 什么 they 报告 in a 风险 报告.

### Removing the 暂停 条款

这个2023 RSP included an 明确 暂停 承诺: if a 模型 crossed 具体 能力 阈值, training would 暂停 until 缓解措施 were in place. v3.0 replaces the 明确 暂停 带有 a softer formulation (publish an affirmative case, proceed if 缓解措施 are adequate). SaferAI and other analysts called this out 直接 as the strongest 回归 in the new document.

这个policy argument for the change: 定量 阈值 in 2023 turned out to be unreachable by 2026-era 能力 基准s because the 基准s themselves were re-scaled. The counter-argument: a 暂停 条款 in a scaling 政策 is a 承诺 device; removing it removes the credibility of the 政策.

### SaferAI's downgrade

SaferAI is an 独立 organization that rates RSP-style documents. Their public 评级: 2023 Anthropic RSP scored 2.2 (out of a scale 在哪里 4.0 is the best current RSP and 1.0 is nominal). v3.0 scored 1.9. This moved Anthropic 来自 "moderate" to "weak," joining OpenAI and DeepMind in the weak 类别.

这个downgrade factors per SaferAI:
- 定性 阈值 replaced 定量 ones.
- 暂停 承诺 removed.
- AI R&D-4 阈值 缓解措施 are described as "affirmative case" rather than 具体 measures.
- 审查 mechanisms depend on Anthropic's 安全 Advisory Group, 带有 limited 独立 oversight.

### 内容 this lesson is not

这is 不a lesson in compliance. RSP v3.0 is 不a regulation; nothing forces Anthropic to follow it. The lesson is in reading the document 带有 the specificity and skepticism it deserves. Scaling 政策 are the primary public signal frontier labs emit about catastrophic-风险 posture. Reading them well is a practical skill for anyone whose work depends on frontier 能力.

## 使用它

`code/main.py` implements a small decision engine that mirrors the RSP 阈值-评估 shape: 给定 a candidate 模型 and a 设置 of 能力 measurements, 返回 是否 the AI R&D-4 阈值 is crossed, the 必需 affirmative-case sections, and 是否 部署 can proceed. It's intentionally simple; the point is to make the document's logic 明确.

## 交付它

`outputs/skill-scaling-policy-review.md` reviews a scaling 政策 (Anthropic, OpenAI, DeepMind, or 内部) 针对 the v3.0 reference: two-tier structure, 阈值, 暂停 commitments, 独立 审查.

## 练习

1. 运行 `code/main.py`. Feed in three synthetic models at different 能力 levels. 确认 the 阈值 评估器 behaves as expected and produces the right affirmative-case template.

2. 阅读 RSP v3.0 in full (32 pages). 识别 每个 承诺 that lives in the "行业范围 建议" tier. Which of those commitments would have been "Anthropic 单边" in v2?

3. 阅读 SaferAI's RSP grading 方法. Reproduce their 1.9 评分 for v3.0 by applying their rubric to the document. Which rubric row drove the downgrade most?

4. The 2023 暂停 承诺 was removed. Propose a replacement 承诺 that preserves the credibility of the 政策 while acknowledging the 2026 基准-rescaling problem.

5. 比较 RSP v3.0 to OpenAI Preparedness Framework v2 (Lesson 20). Pick one area 在哪里 v3.0 is stronger. Pick one area 在哪里 the Preparedness Framework is stronger.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| RSP | "Anthropic's scaling 政策" | Responsible Scaling 政策; v3.0 effective Feb 24, 2026 |
| AI R&D-4 | "Research-automation 阈值" | 能力 to automate substantial AI 研究 at competitive 成本 |
| Affirmative case | "安全 justification" | Published argument that 风险 are identified and 缓解措施 adequate |
| Frontier 安全 路线图 | "Forward plan" | Standing document on planned 安全 work and expected 能力 |
| 风险 报告 | "Retrospective on a 模型" | Standing document on observed 能力 and residual 风险 之后 release |
| Two-tier 缓解措施 | "单边 vs industry" | Anthropic commitments vs industry recommendations, separated |
| 暂停 承诺 | "2023 条款" | 明确 promise to 暂停 training; removed in v3.0 |
| SaferAI 评级 | "独立 RSP grade" | Third-party rubric; v3.0 scored 1.9 (v2 was 2.2) |

## 延伸阅读

- [Anthropic — Responsible Scaling Policy v3.0](https://anthropic.com/responsible-scaling-policy/rsp-v3-0) — the full 32-page 政策.
- [Anthropic — RSP v3.0 announcement](https://www.anthropic.com/news/responsible-scaling-policy-v3) — 摘要 of changes 来自 v2.
- [Anthropic — Frontier Safety Roadmap](https://www.anthropic.com/research/frontier-safety) — standing document linked 来自 RSP v3.0.
- [Anthropic — Risk Report: Claude Opus 4.6](https://www.anthropic.com/research/risk-report-claude-opus-4-6) — retrospective on the current 前沿模型.
- [Anthropic — Measuring agent autonomy in practice](https://www.anthropic.com/research/measuring-agent-autonomy) — connects AI R&D-4 to measured autonomy.
