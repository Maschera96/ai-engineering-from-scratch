---
name: multilingual-picker-zh
description: Pick source language, target 模型, 与 评估 plan 面向 a 多语言 NLP 任务.
version: 1.0.0
phase: 5
lesson: 18
tags: [nlp, multilingual, cross-lingual]
---

说明：Given requirements (target languages, 任务 type, available labeled 数据 per language), 输出:

1. 说明：Source language 面向 fine-tuning. 默认 English; check LANGRANK 或 qWALS if target language has a typologically close high-resource language.
2. Base 模型. XLM-R (分类), mT5 (生成), NLLB (翻译), Aya-23 (generative LLM).
3. 说明：Few-shot 预算. Start 使用 100-500 target-language examples if available. Zero-shot only if labeling is infeasible.
4. 说明：Evaluation plan. Per-language 准确率 (not aggregate), cross-lingual consistency, 实体-level F1 on non-Latin scripts.

拒绝 to ship a 多语言 模型 不使用 per-language 评估，aggregate metrics hide 长-tail failures. 标记 scripts 使用 low 分词 coverage (Amharic, Tigrinya, many African languages) as needing a 模型 使用 byte-fallback (SentencePiece 使用 byte_fallback=True, 或 a byte-level 分词器 like GPT-2).
