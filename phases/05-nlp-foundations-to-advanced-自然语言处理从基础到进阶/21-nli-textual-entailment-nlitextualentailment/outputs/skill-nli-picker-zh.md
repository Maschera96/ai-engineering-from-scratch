---
name: nli-picker-zh
description: Pick an NLI 模型, 标签 template, 与 评估 setup 面向 a 分类 / faithfulness / zero-shot 任务.
version: 1.0.0
phase: 5
lesson: 21
tags: [nlp, nli, zero-shot]
---

给定一个 use case (faithfulness check, zero-shot 分类, 文档-level 推理), 输出:

1. 说明：模型. Named NLI checkpoint. 原因 tied to 领域, length, language.
2. 说明：Template (if zero-shot). Verbalization pattern. Example.
3. 说明：Threshold. Entailment cutoff 面向 the decision rule. 原因 based on calibration.
4. 说明：Evaluation. 准确率 on held-out labeled set, hypothesis-only 基线, adversarial subset.

拒绝 to ship zero-shot 分类 不使用 a 100-example labeled sanity check. 拒绝 to use a 句子-level NLI 模型 on 文档-length premises. 标记 any claim 这 NLI solves hallucination，it reduces it; it does not eliminate it.
