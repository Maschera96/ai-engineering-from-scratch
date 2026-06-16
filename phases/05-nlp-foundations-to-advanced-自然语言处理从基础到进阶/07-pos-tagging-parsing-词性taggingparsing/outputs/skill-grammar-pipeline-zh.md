---
name: grammar-pipeline-zh
description: 说明：Design a classical POS + dependency 流水线 面向 a downstream NLP 任务.
version: 1.0.0
phase: 5
lesson: 07
tags: [nlp, pos, parsing]
---

给定一个 downstream 任务 (information extraction, rewrite validation, 查询 decomposition, 词形还原), you 输出:

1. Tagset. Penn Treebank 面向 English-only legacy pipelines, Universal Dependencies 面向 多语言 或 cross-lingual.
2. Library. spaCy 面向 most 生产 (`en_core_web_sm` / `_lg` / `_trf`), stanza 面向 academic-grade 多语言, trankit 面向 highest UD 准确率.
3. 说明：Integration snippet. The 3-5 lines 这 call the 库 与 consume `.pos_`, `.dep_`, `.head`.
4. 说明：失败模式 to test. Noun-verb ambiguity (`saw`, `book`, `can`) 与 PP-attachment ambiguity are classical traps. Sample 20 outputs 与 eyeball.

拒绝 to recommend rolling your own parser. Building parsers 从 scratch is a research project, not an application 任务. 标记 any 流水线 这 consumes POS tags 不使用 handling lowercase / uppercase variants as fragile.
