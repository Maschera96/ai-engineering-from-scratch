# 大模型功能 A/B 测试：GrowthBook、Statsig 与凭感觉发布问题

> Traditional A/B 测试 was not built for non-deterministic LLMs. 这个关键 distinction: 评估 answer "can 这个模型 do the job?" A/B tests answer "do users care?" Both are 必需; shipping on vibe checks is over. What to test in 2026: 提示词 engineering (wording), 模型 selection (GPT-4 对比 GPT-3.5 对比 OSS; accuracy 对比 成本 对比 延迟), generation parameters (temperature, top-p). Real cases: a chatbot reward-模型 variant delivered +70% conversation length and +30% retention; Nextdoor AI subject-line experiments delivered +1% CTR after reward-function refinement; Khan Academy Khanmigo iterated on 一个延迟-vs-math-accuracy axis. 平台 split: **Statsig** (acquired by OpenAI for $1.1B in September 2025) ： sequential testing, CUPED, all-in-one. **GrowthBook** ： open-source, warehouse-native, Bayesian + Frequentist + Sequential engines, CUPED, SRM checks, Benjamini-Hochberg + Bonferroni corrections. You pick based on warehouse-SQL preference and whether "acquired by OpenAI" matters to your organization.

**Type:** Learn
**Languages:** Python（标准库， 玩具 sequential test模拟器)
**Prerequisites:** Phase 17 · 13 (Observability), Phase 17 · 20 (Progressive Deployment)
**Time:** ~60 分钟

## 学习目标

- Distinguish 评估 ("can 这个模型 do the job") from A/B tests ("do users care").
- Enumerate three testable axes (提示词, 模型, parameters) and pick 这个指标 for each.
- 解释 CUPED, sequential testing, and Benjamini-Hochberg multiple-comparison corrections.
- Pick Statsig or GrowthBook based on warehouse-SQL posture and corporate acquisition stance.

## 问题

You hand-tuned a system 提示词. It feels better. You ship it. Conversion changes by noise. You blame 这个指标. Or you shipped a new 模型 and conversion didn't move ： did 这个模型 degrade or was the change too small to detect? You don't know, because you shipped without an A/B.

评估 answer whether 这个模型 can do a task on a labeled set. They do not answer whether users prefer 这个输出. Only a controlled online experiment answers that, and only if the experiment has enough power, 控制 for non-determinism, and corrects for multiple comparisons.

## 概念

### 评估与 A/B 测试

**评估** ： offline, labeled set, judge (rubric or LLM-as-judge or human). Answer: "Is 这个输出 correct / helpful / safe on this fixed distribution?"

**A/B test** ： online, live users, randomized. Answer: "Does the new variant move the user-level 指标 that matters?"

Both 必需. 评估 catch regressions before exposure; A/B confirms 产品 impact after.

### 测试什么

1. **提示词 engineering** ： wording, system-提示词 structure, examples. 指标: task success, user retention, 成本/请求.
2. **模型 selection** ： GPT-4 对比 GPT-3.5-Turbo 对比 Llama-OSS. 指标: accuracy (task) + 成本/请求 + 延迟 P99. Multi-objective.
3. **Generation parameters** ： temperature, top-p, max_tokens. 指标: task-具体(输出 diversity 对比 determinism).

### CUPED ： 方差 reduction

Controlled-experiments Using Pre-Experiment Data. Regress out pre-period 方差 before comparing post-period. Typical 方差 reduction: 30-70%. Effective sample size goes up for free.

Implementation: both Statsig and GrowthBook implement.

### 序贯测试

Classical A/B assumes fixed sample size. Sequential tests ("peek-and-decide") control false-positive rate under repeated looks. Always-valid sequential procedures (mSPRT, Howard's confidence sequences) let you stop early on clear winners.

### Multiple-comparison corrections

Running 20 A/B tests at 95% confidence 产出 one false positive by chance. Bonferroni correction tightens α per-test; Benjamini-Hochberg 控制 false-discovery rate. GrowthBook implements both.

### SRM ： 样本比例不匹配

Assignment hash randomizes users to variants. If 50/50 split delivers 47/53, something is broken ： SRM check flags it. Both 平台 implement.

### Statsig 对比 GrowthBook

**Statsig**:
- Acquired by OpenAI for $1.1B (September 2025). Hosted, SaaS.
- Sequential testing, CUPED, held-out populations.
- All-in-one: 功能 flags + experimentation + 可观测性.
- Best fit: 团队 already wants a bundled 产品, doesn't care about OpenAI ownership.

**GrowthBook**:
- Open-source (MIT); warehouse-原生(reads from Snowflake/BigQuery/Redshift 直接).
- Multiple engines: Bayesian, Frequentist, Sequential.
- CUPED, SRM, Bonferroni, BH corrections.
- Self-host or managed cloud.
- Best fit: warehouse-SQL shop, data 团队 控制 这个指标 layer, wants OSS.

### Non-determinism complicates power

Same 提示词 产出 varying 输出. Traditional power calculations assume IID observations. With LLM non-determinism, effective sample size is lower than nominal. Multiply 必需 sample size by ~1.3-1.5x as a safety margin.

### Real case outcomes

- Chatbot reward 模型 variant: +70% conversation length, +30% retention.
- Nextdoor subject lines: +1% CTR after reward-function refinement.
- Khan Academy Khanmigo: iterative 延迟-vs-math-accuracy trade.

### The anti-模式: shipping on vibes

Every senior engineer can name 一个功能 that was shipped because "it feels better" with no A/B. Most of them regressed 产品 指标 这个团队 didn't notice for months. A/B is the forcing function.

### 你应该记住的数字

- Statsig acquired by OpenAI: $1.1B, September 2025.
- GrowthBook: open-source MIT; Bayesian + Frequentist + Sequential.
- CUPED 方差 reduction: 30-70%.
- LLM non-determinism → +30-50% sample-size buffer.

## 使用它

`code/main.py` simulates a sequential A/B test with fixed and sequential boundaries. Shows how sequential lets you stop early.

## 交付它

This lesson 产出 `outputs/skill-ab-plan.md`. 给定功能 change, 工作负载, baseline, picks 平台, gates, sample size.

## 练习

1. Run `code/main.py`. For an expected 5% lift with baseline 3% conversion, what sample size to 80% power?
2. Pick Statsig or GrowthBook for a healthcare-regulated on-prem 客户.
3. Design an A/B that tests GPT-4 对比 GPT-3.5 on 成本-per-resolved-ticket. What's 这个主要指标, guardrail 指标, secondary?
4. Your 金丝雀 passes but A/B shows -1.2% conversion. Do you ship? Write the escalation criteria.
5. Apply CUPED to a pre-period with 60% of 这个方差 of post. Compute the effective-sample-size boost.

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| Eval | "offline test" | Labeled-set 评估 of 模型 capability |
| A/B test | "experiment" | Live randomized comparison on users |
| CUPED | "方差 reduction" | Pre-period regression to reduce 方差 |
| Sequential test | "peek-ok test" | Always-valid procedure allowing early stop |
| Multiple comparison | "the family error" | Running many tests inflates false positives |
| Bonferroni | "tight correction" | Divide α by number of tests |
| Benjamini-Hochberg | "BH FDR" | False-discovery-rate control, less conservative |
| SRM | "bad split" | 样本比例不匹配; assignment bug |
| Statsig | "OpenAI owned" | Commercial all-in-one, acquired 2025 |
| GrowthBook | "the OSS one" | MIT warehouse-原生平台 |
| mSPRT | "sequential probability ratio test" | Classical sequential procedure |

## 延伸阅读

- [GrowthBook — How to A/B Test AI](https://blog.growthbook.io/how-to-a-b-test-ai-a-practical-guide/)
- [Statsig — Beyond Prompts: Data-Driven LLM Optimization](https://www.statsig.com/blog/llm-optimization-online-experimentation)
- [Statsig vs GrowthBook comparison](https://www.statsig.com/perspectives/ab-testing-feature-flags-comparison-tools)
- [Deng et al. — CUPED](https://www.exp-platform.com/Documents/2013-02-CUPED-ImprovingSensitivityOfControlledExperiments.pdf)
- [Howard — Confidence Sequences](https://arxiv.org/abs/1810.08240)
