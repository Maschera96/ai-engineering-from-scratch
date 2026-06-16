# OpenAI Preparedness Framework and DeepMind Frontier 安全 Framework

> OpenAI Preparedness Framework v2 (April 2025) introduces Research Categories — Long-range Autonomy, Sandbagging, 自主 Replication and Adaptation, Undermining Safeguards — distinct 来自 Tracked Categories. Tracked Categories 触发器 能力 Reports plus Safeguards Reports reviewed by the 安全 Advisory Group. DeepMind's FSF v3 (September 2025, 带有 Tracked 能力 Levels added April 17, 2026) folds autonomy 进入 ML R&D and Cyber domains (ML R&D autonomy level 1 = fully automate the AI R&D 流水线 at competitive 成本 vs 人类 + AI 工具). FSF v3 explicitly addresses deceptive 对齐 via automated 监控 for instrumental-reasoning misuse. The honest note: Research Categories in PF v2 (including Long-range Autonomy) do 不automatically 触发器 缓解措施; the 政策 language is "potential." DeepMind itself says automated 监控 "will 不remain sufficient long-term" if instrumental reasoning strengthens.

**Type:** Learn
**Languages:** Python (stdlib, three-framework decision-表格 diff 工具)
**Prerequisites:** Phase 15 · 19 (Anthropic RSP)
**Time:** ~45 minutes

## 问题

Lesson 19 阅读 Anthropic's scaling 政策 closely. This lesson completes the picture by reading OpenAI's and DeepMind's. The three documents are cousin artifacts addressing the same question — when 应该 a frontier lab 暂停 or gate a 模型 — and they converge on a small 设置 of 类别 and diverge in 具体 places that matter.

这个convergence: 所有 three label long-range autonomy as a 能力 类别 worth tracking. 所有 three acknowledge deceptive behavior (对齐 faking, sandbagging) as a 具体 类别 of 风险. 所有 three have an 内部 审查 body. The divergence: OpenAI splits 类别 进入 "Tracked" (mandatory 缓解措施) and "Research" (no automatic 触发器). DeepMind folds autonomy 进入 two domains rather than naming it separately. The lab names Tracked vs Research, or Critical vs Moderate, or Tier-1 vs Tier-2; the operational consequence of 哪个 bucket a 能力 lives in is different across labs.

Reading them together is the useful exercise. The same 能力 can be "mandatory 缓解措施" at Anthropic, "monitored but 不triggering" at OpenAI, and "tracked in a 具体 domain" at DeepMind. The 政策 posture matters.

## 概念

### OpenAI Preparedness Framework v2 (April 2025)

Structure:

- **Tracked Categories**: 触发器 能力 Reports (什么 the 模型 can do) plus Safeguards Reports (什么 缓解措施 are in place). Reviewed by the 安全 Advisory Group 之前 部署.
- **Research Categories**: 模型 能力 the lab is tracking but 不yet committing to 具体 缓解措施. Includes Long-range Autonomy, Sandbagging, 自主 Replication and Adaptation, Undermining Safeguards.

Key framing in v2: Research Categories do 不automatically 触发器 缓解措施. The 政策 language is "potential" 缓解措施. This is the 具体 delta to watch: a 类别's 位置 in Tracked vs Research is the hinge between "gated" and "observed."

### DeepMind Frontier 安全 Framework v3 (September 2025; Tracked 能力 Levels added April 17, 2026)

Structure:

- **Critical 能力 Levels (CCLs)**: 能力 阈值 across five domains: Cyber, Bio, ML R&D, CBRN, Autonomy (folded 进入 ML R&D and Cyber).
- **Tracked 能力 Levels**: additional granularity added in April 2026. 具体 example: ML R&D autonomy level 1 = fully automate the AI R&D 流水线 at competitive 成本 vs 人类 + AI 工具.
- **Deceptive 对齐 监控**: 明确 承诺 to automated 监控 for instrumental-reasoning misuse.

这个autonomy framing differs 来自 OpenAI's. DeepMind does 不keep "Autonomy" as a top-level domain; it is folded 进入 the domains 在哪里 autonomy would cause harm (ML R&D and Cyber). The argument is that autonomy 没有 a domain is 能力 没有 风险; the counter-argument is that autonomy across domains is a meta-风险 the framework 应该 命名.

### 内容 所有 three converge on

- 内部 安全 Advisory Group (具名 Anthropic SAG, OpenAI SAG, DeepMind 内部 committee). 审查 之前 部署 for high-能力 models.
- 明确 mention of deceptive 对齐 / 对齐 faking as a 风险 类别.
- Standing artifacts on a declared 节奏 (Anthropic: Frontier 安全 路线图, 风险 报告; OpenAI: 能力 and Safeguards Reports; DeepMind: FSF update cycle).
- Acknowledgement that 监控-只 防御 have a ceiling. DeepMind is 明确: "automated 监控 will 不remain sufficient long-term."

### 位置 they diverge

- **Anthropic**: 暂停 承诺 removed in v3.0; AI R&D-4 阈值 is the 具名 next gate.
- **OpenAI**: Tracked vs Research split; Research Categories (including Long-range Autonomy) do 不automatically gate.
- **DeepMind**: autonomy folded 进入 other domains; Tracked 能力 Levels add granularity in April 2026.

### Sandbagging: a 具体 能力 that complicates 所有 three

Sandbagging (a 模型 strategically underperforming on 评估s) is in OpenAI's Research Categories. Anthropic's RSP v3.0 addresses it via the 评估-上下文 缺口 (Lesson 1). DeepMind addresses it via deceptive 对齐 监控 in FSF v3.

如果a 模型 sandbags on 评估s, 每个 framework's 能力 阈值 are underestimated. The framework works 只 if the measurement works. This is 为什么 外部 measurement (Lesson 21, METR) and adversarial 评估 are necessary in addition to lab self-assessment.

### 这个policy-reading skill

- Locate: 每个 能力 you care about 应该 be findable in the 政策. If 不findable, the 政策 does 不cover it.
- Classify: is it Tracked (triggers 缓解措施) or Research (tracked but 不triggering)? OpenAI names this; Anthropic and DeepMind have their own equivalents.
- 节奏: is the 政策 updated on a declared schedule, or 只 之后 具体 events? Declared 节奏 is stronger.
- Independence: is 外部 审查 mandatory or optional? Anthropic partners 带有 Apollo and US AI 安全 Institute; OpenAI 带有 METR; DeepMind 带有 内部 SAG primarily.

## 使用它

`code/main.py` implements a small decision-表格 diff 工具. 给定a 能力 (autonomy, deceptive 对齐, R&D automation, cyber uplift, etc.), it 输出 如何 each of the three 政策 classifies the 能力, and 什么 缓解措施 触发器. It's a reading aid, 不a 政策 工具.

## 交付它

`outputs/skill-cross-policy-diff.md` produces a cross-政策 comparison for a 具体 能力, using the three frameworks as reference.

## 练习

1. 运行 `code/main.py`. 确认 the diff 工具's 输出 matches the 政策 for at least two 能力 you can 验证 针对 the 来源 documents.

2. 阅读 OpenAI Preparedness Framework v2 in full. 识别 each Research Category. For each, 编写 一句话 on 为什么 it is in Research rather than Tracked.

3. 阅读 DeepMind FSF v3 in full, plus the April 2026 Tracked 能力 Levels update. 识别 ML R&D autonomy level 1's 具体 评估 criteria. 方式 would you measure it externally?

4. Sandbagging is in OpenAI's Research Categories. 设计 an 评估 that would force a sandbagging 模型 to reveal its actual 能力. Reference the Lesson 1 评估-上下文-gaming discussion.

5. 比较 the three 政策 on a 具体 能力 (your choice). 命名 哪个 政策's 分类 you find most rigorous and 哪个 least. Justify 带有 来源 text.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| Preparedness Framework | "OpenAI's scaling 政策" | PF v2 (April 2025); Tracked vs Research 类别 |
| Tracked Category | "Mandatory 缓解措施" | Triggers 能力 + Safeguards Reports; SAG 审查 |
| Research Category | "Monitored 只" | Tracked but no automatic 缓解措施; includes Long-range Autonomy |
| Frontier 安全 Framework | "DeepMind's scaling 政策" | FSF v3 (Sept 2025) + Tracked 能力 Levels (Apr 2026) |
| CCL | "Critical 能力 Level" | DeepMind 阈值 per domain (Cyber, Bio, ML R&D, CBRN) |
| ML R&D autonomy level 1 | "R&D automation" | Fully automate AI R&D 流水线 at competitive 成本 |
| Sandbagging | "Strategic underperformance" | Model underperforms on evals; in OpenAI Research Categories |
| Instrumental reasoning | "Means-ends reasoning" | Reasoning about 如何 to achieve goals; target of DeepMind 监控 |

## 延伸阅读

- [OpenAI — Updating our Preparedness Framework](https://openai.com/index/updating-our-preparedness-framework/) — v2 announcement.
- [OpenAI — Preparedness Framework v2 PDF](https://cdn.openai.com/pdf/18a02b5d-6b67-4cec-ab64-68cdfbddebcd/preparedness-framework-v2.pdf) — full document.
- [DeepMind — Strengthening our Frontier Safety Framework](https://deepmind.google/blog/strengthening-our-frontier-safety-framework/) — FSF v3 announcement.
- [DeepMind — Updating the Frontier Safety Framework (April 2026)](https://deepmind.google/blog/updating-the-frontier-safety-framework/) — Tracked 能力 Levels addition.
- [Gemini 3 Pro FSF Report](https://storage.googleapis.com/deepmind-media/gemini/gemini_3_pro_fsf_report.pdf) — example of an FSF-format 风险 报告.
