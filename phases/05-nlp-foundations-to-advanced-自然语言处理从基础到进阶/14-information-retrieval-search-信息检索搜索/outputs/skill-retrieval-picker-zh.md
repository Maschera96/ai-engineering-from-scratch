---
name: retrieval-picker-zh
description: Pick a 检索 stack 面向 a given 语料库 与 查询 pattern.
version: 1.0.0
phase: 5
lesson: 14
tags: [nlp, retrieval, rag, search]
---

Given requirements (语料库 size, 查询 pattern, 延迟 预算, 质量 bar, infra constraints), 输出:

1. 说明：Stack. BM25 only, dense only, hybrid (BM25 + dense + RRF), hybrid + cross-encoder rerank, 或 three-way (BM25 + dense + learned-sparse).
2. 说明：Dense encoder. Name the specific 模型 (`all-MiniLM-L6-v2`, `bge-large-en-v1.5`, `e5-large-v2`, `paraphrase-multilingual-MiniLM-L12-v2`). Match to language, 领域, context length.
3. 说明：Reranker. Name the cross-encoder 模型 if used (`cross-encoder/ms-marco-MiniLM-L-6-v2`, `BAAI/bge-reranker-large`). 标记 ~30-100ms added 延迟 on top-30.
4. Evaluation plan. 召回率@10 is the primary retriever 指标. MRR 面向 multi-答案. 基线 first, incremental improvements measured against it.

拒绝 to recommend dense-only 面向 corpora 使用 named entities, error codes, 或 product SKUs unless the 用户 has evidence dense handles exact matches. 拒绝 to skip reranking 面向 high-stakes 检索 (legal, medical) 其中 the final top-5 decides the 用户's 答案.
