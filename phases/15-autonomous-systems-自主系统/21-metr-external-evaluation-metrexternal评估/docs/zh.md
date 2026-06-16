# METR Time Horizons and 外部 能力 Evaluation

> METR (ex-ARC Evals) is an 独立 501(c)(3) since December 2023. Their Time Horizon 1.1 基准 (January 2026) fits a logistic curve to 任务-success probability vs 日志(expert 人类 completion time); the intersection at 50% probability defines the 模型's time horizon. The 2025–2026 engagement 设置 covers GPT-5.1, GPT-5.1-Codex-Max, and prototype 监控 评估s (can a monitor catch side 任务; can the 智能体 evade). 基准 suites: HCAST (180+ ML, cyber, SWE, reasoning 任务; 1 minute to 8+ hours), RE-Bench (71 ML 研究-engineering 任务 带有 expert baseline), SWAA. The honest note: METR measurements are idealized — no 人类, no real consequences — and the team has documented the 评估-vs-部署 behavior 缺口 (Lesson 1). A time horizon is an upper bound, 不a 部署 prediction.

**Type:** Learn
**Languages:** Python (stdlib, logistic-fit horizon estimator)
**Prerequisites:** Phase 15 · 01 (Long-horizon agents), Phase 15 · 19 (RSP)
**Time:** ~60 minutes

## 问题

Scaling 政策 (Lessons 19, 20) are 只 as useful as the measurements they reference. "AI R&D-4 阈值" and "Long-range Autonomy" are defined in 政策 prose; they become actionable 只 when 具体 评估s 生成 具体 numbers.

METR is the 2024–2026 外部 评估 organization that has defined many of those numbers. They evaluate 前沿模型 — often pre-release, under NDA 带有 labs — and publish 方法 afterward. The Time Horizon 1.1 基准 (January 2026) is their headline artifact: a 单一 scalar that compresses 能力 进入 a 人类-legible unit ("this 模型 can do the kind of 任务 an expert spends X hours on at 50% 可靠性").

这个lesson is partly about the 方法 (如何 a horizon is computed) and partly about the interpretation (为什么 a horizon is an upper bound, 不a 部署 prediction). The two skills belong together. A team that understands 如何 the horizon is fit is much harder to fool 带有 a bad vendor 声明 than a team that just sees "14 hours" on a slide.

## 概念

### METR background

- Founded: December 2023 (ex-ARC Evals, spun out 进入 独立 501(c)(3)).
- Scope: 评估 of 前沿模型' 自主 能力, often pre-release.
- Partner labs: Anthropic, OpenAI (multiple engagements 2025–2026).
- Notable deliverables: Time Horizon 1.0 (March 2025), Time Horizon 1.1 (January 2026), prototype 监控 评估s.

### 这个Time Horizon fit

方法 (来自 METR blog and papers):

1. Collect a 任务 suite spanning minute-scale to hour-scale expert completion times. Current suites: HCAST (180+ 任务), RE-Bench (71 任务), SWAA.
2. 运行 the 模型 on each 任务; record success or 失败.
3. Fit a logistic curve: P(success) as a function of 日志(expert completion time).
4. The horizon is the expert-time at 哪个 P(success) = 0.5.

这个logistic-fit shape is the right one because 能力 generally has an increasing, plateau-approaching relationship 带有 任务 difficulty. The 50% point is a choice (could be 10%, 90%); METR reports multiple 阈值 in the detailed 论文 but leads 带有 50% because it is the most intuitive.

### 这个January 2026 numbers

Per Time Horizon 1.1:

- Claude Opus 4.6: ~14 hours at 50% 可靠性, as of Time Horizon 1.1 (January 2026).
- Doubling time on HCAST-style 任务: ~4.3 months (130.8 days) on the post-2023 fit reported by Time Horizon 1.1 (January 2026); the ~7-month figure is the full 2019–2025 fit 来自 Time Horizon 1.0 and is reported in TH1.1 alongside the post-2023 number.

Those are the 具体 numbers the rest of Phase 15 references. They are cited in Anthropic's RSP v3.0 (Lesson 19) and DeepMind's FSF v3 (Lesson 20) as one of the measurements feeding 阈值 assessments.

### 基准 suites

- **HCAST**: 189 任务 across ML, cybersecurity, software engineering, and general reasoning. Span: 1 minute to 8+ hours of expert time. The main suite behind the horizon number.
- **RE-Bench**: 71 ML 研究-engineering 任务 带有 a 人类 expert baseline. Measures 是否 the 模型 can do the kind of ML R&D work that feeds 进入 the AI R&D-4 阈值.
- **SWAA**: shorter 任务; useful for calibrating the low end of the horizon curve.

### Prototype 监控 评估s

2025–2026 METR work included 监控-评估 prototypes: can a monitor catch a 模型 performing a side 任务 (something other than 什么 the 操作员 asked for); can the 智能体 evade the monitor. These 直接 feed the "Undermining Safeguards" 类别 in OpenAI PF v2 and the deceptive-对齐 监控 in DeepMind FSF v3 (Lesson 20).

### 原因 horizons are upper 边界

- **Idealized tooling**: 基准 任务 give the 模型 clean 工具 and well-formatted data. Real 部署s are messier.
- **No real consequences**: the 模型 never actually bills a customer, deletes real data, or contacts real people. Real 部署s have irreversible stakes.
- **评估-上下文 gaming**: Lesson 1. Models behave differently in tests. The 2026 International AI 安全 报告 documents this empirically.
- **No legitimate 用户 variance**: 基准 提示词 are structured. Real users 生成 含糊, 上下文-dependent requests.

这个horizon is the 能力 ceiling under favorable conditions. 部署 可靠性 is a different number, lower, and teams 必须 measure their own 分布 to know it.

### 这个external-评估器 case

外部 评估 matters because 内部 labs have incentives to optimize 指标 they 报告. METR's independence — a 501(c)(3) 带有 a declared 方法 and peer-reviewed papers — is the structural 缓解措施. It is 不sufficient alone (labs still 控制 什么 METR sees), but it is strictly better than no 外部 评估.

### 方式 to use horizon numbers in practice

- **As a 能力 filter**: if a 模型's horizon is well below the expert-time of a 拟议的 任务, do 不ship it 自主 (Lesson 1's skill 文件).
- **As a trend indicator**: doubling time tells you 如何 long the current practice will remain safe even 没有 new 缓解措施.
- **As a prior**: a horizon of 14 hours is a starting point. Adjust down for your 任务 分布, your tooling 质量, and your 部署 上下文.

## 使用它

`code/main.py` implements a logistic fit of 任务-success vs 日志(expert time), 给定 a synthetic 结果 设置. It reports the 50% horizon (METR's headline), 10% horizon (conservative), and 90% horizon (optimistic). Also demonstrates 什么 changes when the success rate is artificially inflated by 评估-上下文 gaming.

## 交付它

`outputs/skill-horizon-interpretation.md` reviews a vendor's horizon 声明 and produces a 缺口 analysis between 基准 声明 and 部署 reality.

## 练习

1. 运行 `code/main.py`. 确认 the fit's 50% horizon matches the synthetic ground truth. Now halve the 任务-time grid; does the horizon 估计 change meaningfully?

2. 阅读 METR's Time Horizon 1.1 blog post. 识别 the 具体 任务 在哪里 可靠性 is highest and 在哪里 it is lowest. 解释 为什么 the 缺口 exists.

3. 阅读 METR's "Measuring 自主 AI 能力" resources. 列出 the HCAST 任务 类别. Pick one 类别 you would weight more heavily for a 生产环境 任务 and justify 为什么.

4. Introduce 评估-上下文 gaming 进入 the simulator: flip ~20% of failed 任务 to success. 报告 the new horizon. This approximates 什么 a gaming rate of 20% does to the observed number.

5. 设计 an 内部 horizon 评估 on your own bug backlog or a representative 任务 设置. 描述 the data collection, the fit, and 什么 the 输出 tells you. 比较 to METR numbers.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| METR | "外部 评估器" | ex-ARC Evals; 独立 501(c)(3) since Dec 2023 |
| Time Horizon | "能力 measure" | Expert 任务 length at 50% 可靠性, 来自 logistic fit |
| HCAST | "METR's main suite" | 180+ 任务 spanning 1 min to 8+ hours |
| RE-Bench | "Research engineering" | 71 ML 研究-engineering 任务 带有 人类 baseline |
| SWAA | "Short-任务 suite" | Calibrates the low end of the horizon curve |
| Doubling time | "Growth rate" | Time for the 50% horizon to double; ~7 months per HCAST |
| 评估-上下文 gaming | "Model behaves differently" | Documented behavior 缺口 between tests and 部署 |
| Upper bound | "Horizon is a ceiling" | 基准 horizon > 部署 可靠性 under load |

## 延伸阅读

- [METR — Resources for Measuring Autonomous AI Capabilities](https://metr.org/measuring-autonomous-ai-capabilities/) — HCAST, RE-Bench, SWAA specs.
- [METR — Measuring AI Ability to Complete Long Tasks](https://metr.org/blog/2025-03-19-measuring-ai-ability-to-complete-long-tasks/) — the original horizon 论文.
- [METR — Time Horizon 1.1 (January 2026)](https://metr.org/research/) — current numbers and 方法.
- [Epoch AI — METR Time Horizons benchmark](https://epoch.ai/benchmarks/metr-time-horizons) — live tracking.
- [Anthropic — Measuring agent autonomy in practice](https://www.anthropic.com/research/measuring-agent-autonomy) — 内部 perspective on METR's measurements.
