---
name: entity-linker-zh
description: 说明：Design an 实体 linking 流水线，KB, candidate generator, disambiguator, 评估.
version: 1.0.0
phase: 5
lesson: 25
tags: [nlp, entity-linking, knowledge-graph]
---

给定一个 use case (领域 KB, language, volume, 延迟 预算), 输出:

1. 说明：Knowledge base. Wikidata / Wikipedia / custom KB. Version date. Refresh cadence.
2. 说明：Candidate generator. Alias-index, 嵌入, 或 hybrid. Target mention 召回率 @ K.
3. 说明：Disambiguator. Prior + context, 嵌入-based, generative, 或 LLM-prompted.
4. 说明：NIL strategy. Threshold on top score, classifier, 或 explicit NIL candidate.
5. 说明：Evaluation. Mention 召回率 @ 30, top-1 准确率, NIL-detection F1 on held-out set.

拒绝 any EL 流水线 不使用 a mention-召回率 基线 (you cannot evaluate a disambiguator 不使用 knowing candidate gen surfaced the right 实体). 拒绝 any 流水线 using LLM-prompted EL 不使用 constrained 输出 to 有效 KB ids. 标记 systems 其中 popularity bias affects minority entities (e.g. name-clashes) 不使用 领域 fine-tuning.
