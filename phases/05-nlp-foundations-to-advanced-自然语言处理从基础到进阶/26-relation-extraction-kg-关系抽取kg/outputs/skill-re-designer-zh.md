---
name: re-designer-zh
description: Design a 关系 extraction 流水线 使用 provenance 与 canonicalization.
version: 1.0.0
phase: 5
lesson: 26
tags: [nlp, relation-extraction, knowledge-graph]
---

给定一个 语料库 (领域, language, volume) 与 downstream use (KG-RAG, analytics, compliance), 输出:

1. Extractor. Pattern-based / supervised / LLM / AEVS hybrid. 原因 tied to 精确率 vs 召回率 target.
2. Ontology. 封闭 property list (Wikidata / 领域) 或 开放 IE 使用 canonicalization pass.
3. 说明：Provenance. Every triple carries source char-span + doc id. Non-negotiable 面向 audit.
4. 说明：Merge strategy. Canonical 实体 id + 关系 id + temporal qualifiers; dedup policy.
5. 说明：Evaluation. 精确率 / 召回率 on 200 hand-labelled triples + hallucination-rate on LLM-extracted sample.

拒绝 any LLM-based RE 流水线 不使用 span verification (source provenance). 拒绝 开放-IE 输出 flowing 到 a 生产 graph 不使用 canonicalization. 标记 pipelines 使用 no temporal qualifier on time-bounded relations (employer, spouse, position).
