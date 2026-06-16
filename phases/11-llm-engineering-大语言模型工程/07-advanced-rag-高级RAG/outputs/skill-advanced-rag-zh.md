---
name: skill-advanced-rag-zh
description: 构建production-grade RAG with hybrid search, 重排, and 评估
version: 1.0.0
phase: 11
lesson: 7
tags: [rag, hybrid-search, bm25, reranking, hyde, evaluation]
---

# Advanced RAG Pattern

Basic RAG: embed 查询 -> vector search -> top-k -> 生成.
Advanced RAG: embed 查询 + BM25 -> fuse ranks -> rerank -> top-k -> 生成.

```text
query -> [vector search (top-50)] -+-> RRF fusion -> reranker (top-5) -> prompt -> LLM
                                   |
query -> [BM25 search (top-50)]  --+
```

## When to upgrade from basic RAG

- 检索 质量 drops below 70% Recall@5
- Users report wrong or irrelevant answers
- 语料库 grows beyond 100K chunks
- 查询 use different 词表 than 文档
- Multi-hop 问题 fail consistently

## Implementation checklist

1. Add BM25 index alongside vector index
2. 运行both searches in 并行 (top-50 each)
3. Merge with Reciprocal 排序 Fusion (k=60)
4. Rerank top candidates with a cross-encoder
5. Take top-5 for the final 提示词
6. Add faithfulness 评估 on a 测试集

## Technique selection guide

- **Hybrid search**: always use in 生产. 成本 nothing extra at 查询 time.
- **重排**: use when Recall@50 is good but Recall@5 is bad. Adds 50-200ms 延迟.
- **HyDE**: use when 查询 are vague or use different 词表 than docs. Adds one LLM call.
- **Parent-child chunks**: use when small chunks lack 上下文 but large chunks dilute relevance.
- **Metadata filtering**: use when 语料库 has clear categories (date, 来源 type, department).
- **查询 decomposition**: use for multi-hop 问题 that require information from multiple docs.

## Common mistakes

- Running BM25 and vector search with different 分块 sets (they must search the same 语料库)
- Using too small a candidate pool for 重排 (top-10 is too few; use top-50)
- Adding HyDE for every 查询 (only helps when 词表 mismatch is the bottleneck)
- Not evaluating changes (measure Recall@k before and after each technique)
- Over-engineering the 流水线 before measuring where it fails

## 评估 工作流

1. Create 50+ test 问题 with known 答案 chunks
2. Measure Recall@5 and Recall@10 for each 检索 method
3. For 查询 where 检索 succeeds, measure faithfulness of 生成的 answers
4. Track 指标 weekly as the 语料库 grows
5. Investigate individual failures before adding more techniques
