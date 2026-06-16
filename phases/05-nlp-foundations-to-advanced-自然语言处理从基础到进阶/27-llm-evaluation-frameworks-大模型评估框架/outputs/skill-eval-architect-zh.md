---
name: eval-architect-zh
description: 说明：Design an LLM 评估 plan 使用 calibrated judge 与 CI gates.
version: 1.0.0
phase: 5
lesson: 27
tags: [nlp, evaluation, rag]
---

给定一个 use case (RAG / agent / generative 任务), 输出:

1. Metrics. Faithfulness / relevance / context-精确率 / context-召回率 + any custom G-Eval metrics 使用 criteria.
2. Judge 模型. Named 模型 + version, rationale 面向 成本 vs 准确率.
3. 说明：Calibration. Hand-labeled set size, target Spearman rho vs human > 0.7.
4. 说明：Dataset versioning. Tag strategy, change log, stratification.
5. 说明：CI gate. Thresholds per 指标, regression-window logic, bottom-quantile alert.

拒绝 to rely on a judge untested against ≥50 human-labeled examples. 拒绝 self-评估 (same 模型 generates + judges). 拒绝 aggregate-only reporting 不使用 bottom-10% surfacing. 标记 any 流水线 其中 judge upgrade lands 不使用 parallel 基线 eval.
