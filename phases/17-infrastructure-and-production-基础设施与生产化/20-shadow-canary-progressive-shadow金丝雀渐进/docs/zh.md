# 大模型的影子流量、金丝雀发布与渐进式部署

> LLM rollouts combine the hardest parts of software 部署: no unit tests, diffuse failure modes, delayed signals. The sequence is (1) shadow mode ： duplicate prod 请求 to candidate 模型, log, 比较 with zero user impact; catches obvious distribution issues but is not 一个质量 guarantee; (2) 金丝雀 rollout ： 渐进式 流量 shift 10% → 25% → 50% → 75% → 100% with gates at each 步骤; track 延迟 percentiles, 成本/请求, error/refusal rate, 输出 length distribution, user-feedback rate; (3) A/B 测试 for distinct alternatives after stability confirmed. Non-determinism is irreducible ： up to 15% accuracy variation across runs with identical 输入 due to GPU FP non-associativity plus 批处理-size 方差. 成本 is a variable, not constant ： a 20% better 模型 can be 3x more expensive per call. 回滚 speed is decisive: if 回滚 需要 redeploy, you are too slow. 策略 lives in config/flags; 模型 lives in registry with pinned digests; 回滚 = flip 策略 + revert threshold + pin old 模型 in seconds.

**Type:** Learn
**Languages:** Python（标准库， 玩具 canary-progression模拟器)
**Prerequisites:** Phase 17 · 13 (Observability), Phase 17 · 21 (A/B Testing)
**Time:** ~60 分钟

## 学习目标

- Distinguish shadow mode (zero-impact 比较), 金丝雀 (live 流量 渐进式), and A/B (stability-confirmed comparison).
- Enumerate five LLM-具体金丝雀 指标 (延迟, 成本/请求, error/refusal, 输出-length distribution, user feedback).
- 解释 why LLM non-determinism (up to 15%) changes what "stable" means in a rollout.
- Design 一个回滚 path that takes seconds (策略 flip) not hours (redeploy).

## 问题

You ship a new 模型. Offline 评估 show 3% accuracy gain. You flip it on in 生产化. Within 24 hours, 成本 is up 40%, user thumbs-down is up 8%, three 客户 tickets report "weird answers." You roll back. Redeploy takes 3 hours. Your weekend is ruined.

Every piece of that was avoidable. Shadow mode would have caught the 40% 成本 spike before any user saw it. 金丝雀 would have stopped at 10% when thumbs-down moved. 策略-flag 回滚 would have taken 30 seconds. The discipline is what fills in the gap between "offline 评估 look good" 和 "real users are happy."

## 概念

### 影子模式

Candidate receives the same 请求 as 生产化; 输出 are logged, not returned to users. Zero user impact. Log:

- 输出 content (diff against 生产化).
- 词元 counts (成本 delta).
- 延迟.
- Refusal and error.

Catches: 成本 blow-ups, length regressions, obvious refusal changes, hard errors. Does NOT catch: 质量 delta users would perceive. Shadow is a smoke test, not 一个质量 test.

### 金丝雀发布

渐进式 流量 shift with gates. Typical progression: 1% → 10% → 25% → 50% → 75% → 100%. Gate on 5 指标 at each 步骤:

1. **延迟 percentiles** ： P50, P95, P99. Breach: 金丝雀 has P99 > 1.5x baseline.
2. **成本 per 请求** ： blended $. Breach: >20% above baseline.
3. **Error / refusal rate** ： 5xx plus explicit refusals. Breach: 2x baseline.
4. **输出 length distribution** ： 均值 + P99. Breach: distributional shift.
5. **User-feedback rate** ： thumbs-down / ticket filings. Breach: 1.5x baseline.

### 非确定性是新的方差

Identical 输入 produce non-identical 输出. Reasons:

- GPU FP non-associativity (floating-point reduction order 变化 by 批处理).
- 批处理-size 方差 (same 提示词 in 一个批处理 of 128 对比 批处理 of 16).
- Sampling (temperature > 0).

Measured: up to 15% accuracy variation run-to-run on identical eval sets. "Stable" in a rollout means 指标 are within expected 方差, not identical to baseline. Set gates above the noise floor.

### 成本是变量

A 20% better 模型 can be 3x more expensive per call. 成本/请求 is one of the five gates. Shipping 一个"better" 模型 that breaks 单位经济学 is 一个回滚 case.

### 回滚是武器

- 策略 flag (功能 flag system): flip percentage in config; takes seconds.
- 模型 pinning (registry digest): pinned 模型 does not auto-upgrade.
- 回滚 = revert flag + set pinned digest to previous. Seconds, not hours.

如果your stack 需要 redeploy to 回滚, fix that before rolling.

### 工具链

**Argo Rollouts** / **Flagger** ： Kubernetes 渐进式 delivery controllers. Integrate with Istio/Linkerd weighted 路由.

**Istio weighted 路由** ： service-mesh-level 流量 split.

**KServe / Seldon Core** ： 模型 serving with built-in 金丝雀.

**功能 flags** ： LaunchDarkly, Flagsmith, Unleash. 策略-level flip, no redeploy.

### 指标节奏

金丝雀 gates check every 5-15 minutes depending on 流量 volume. 1% 流量 with 10 req/min gives 50-150 data points per window ： enough for 延迟 but noisy for user feedback. 10% gives ~10x more. Progressions should pause long enough to accumulate enough samples at each 步骤.

### The A/B 步骤 is optional

如果the new 模型 is distinctly 不同(different behavior, 不同成本 curve, different tone), A/B test it at 50% after 金丝雀 passes. If it's just an improved version, skip to 100% 什么时候 金丝雀 gates pass.

### 你应该记住的数字

- 金丝雀 progression: 1% → 10% → 25% → 50% → 75% → 100%.
- Non-determinism ceiling: up to 15% run-to-run 方差 on identical 输入.
- Five 金丝雀 指标: 延迟, 成本, error/refusal, 输出 length, user feedback.
- 成本 gate: >20% above baseline is a breach.
- 回滚: seconds, not hours.

## 使用它

`code/main.py` simulates 一个金丝雀 rollout with injected regressions. Reports which stage the rollout halts at and which gate triggered.

## 交付它

This lesson 产出 `outputs/skill-rollout-runbook.md`. Given candidate 模型, baseline, and 风险 tolerance, designs shadow→金丝雀→100% 计划.

## 练习

1. Run `code/main.py`. Inject a 25% 成本 regression. At which stage does 这个金丝雀 halt?
2. Your new 模型 has 3% accuracy gain offline but 成本/请求 is +18%. Is it a ship? Depends on 这个策略 ： write both paths.
3. Design 一个回滚 that takes under 60 seconds end-to-end. List 这个必需的基础设施.
4. Non-determinism shows ±7% on your eval. Set 金丝雀 gates so you don't false-alarm. What multipliers do you use?
5. Shadow mode catches a 40% 成本 spike before 金丝雀. Write the alert rule that fires in shadow.

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| Shadow mode | "duplicate to new" | Zero-impact send-to-candidate for logging |
| 金丝雀 | "渐进式 流量" | Gradual user-exposed rollout with gates |
| Gates | "rollout checks" | 指标 thresholds that block progression |
| Non-determinism | "LLM 方差" | Irreducible run-to-run differences |
| 策略 flag | "flag flip 回滚" | Config-level 回滚, seconds not hours |
| 模型 pin | "registry digest" | Immutable reference to 一个模型 version |
| Argo Rollouts | "K8s 渐进式" | Kubernetes-原生金丝雀/回滚 controller |
| KServe | "inference K8s" | 模型 serving with 金丝雀 primitives |
| Istio weighted | "mesh split" | Service-mesh 流量 splitter |

## 延伸阅读

- [TianPan — Releasing AI Features Without Breaking Production](https://tianpan.co/blog/2026-04-09-llm-gradual-rollout-shadow-canary-ab-testing)
- [MarkTechPost — Safely Deploying ML Models](https://www.marktechpost.com/2026/03/21/safely-deploying-ml-models-to-production-four-controlled-strategies-a-b-canary-interleaved-shadow-testing/)
- [APXML — Advanced LLM Deployment Patterns](https://apxml.com/courses/mlops-for-large-models-llmops/chapter-4-llm-deployment-serving-optimization/advanced-llm-deployment-patterns)
- [Argo Rollouts docs](https://argo-rollouts.readthedocs.io/)
- [Flagger docs](https://docs.flagger.app/)
