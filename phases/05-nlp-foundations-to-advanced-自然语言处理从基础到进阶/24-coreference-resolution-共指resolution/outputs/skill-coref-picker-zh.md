---
name: coref-picker-zh
description: 说明：Pick a coreference approach, 评估 plan, 与 integration strategy.
version: 1.0.0
phase: 5
lesson: 24
tags: [nlp, coref, information-extraction]
---

给定一个 use case (single-doc / multi-doc, 领域, language), 输出:

1. 说明：Approach. Rule-based / neural span-based / LLM-prompted / hybrid. One-句子 原因.
2. 模型. Named checkpoint if neural.
3. 说明：Integration. Order of operations: tokenize → NER → coref → downstream 任务.
4. 说明：Evaluation. CoNLL F1 (MUC + B³ + CEAF-φ4 average) on held-out set + manual 聚类 review on 20 文档.

拒绝 LLM-only coref 面向 文档 over 2,000 词元 不使用 sliding-window merge. 拒绝 any 流水线 这 runs coref 不使用 a mention-level 精确率-召回率 report. 标记 gender-heuristic systems deployed in demographically diverse 文本.
