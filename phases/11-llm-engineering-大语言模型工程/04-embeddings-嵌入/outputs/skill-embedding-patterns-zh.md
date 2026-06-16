---
name: skill-embedding-patterns-zh
description: 生产 patterns for 嵌入s, vector search, and 相似度
version: 1.0.0
phase: 11
lesson: 4
tags: [embeddings, vectors, similarity, search, chunking, quantization]
---

# 嵌入 Patterns

每个嵌入 工作流 follows this contract:

```text
text -> embed(text) -> vector (float array)
similarity(vector_a, vector_b) -> score (float)
```

这个嵌入 模型 and 相似度 指标 are the only two decisions that matter. Everything else is plumbing.

## When to use 嵌入s

- 语义 search across 文档 (find meaning, not keywords)
- Clustering similar items (support tickets, product reviews, bug reports)
- 分类 by nearest neighbors (标签 new items by 相似度 to 标签ed examples)
- Recommendation systems (find items similar to what the 用户 liked)
- Deduplication (find near-duplicate content using 相似度 阈值)

## When NOT to use 嵌入s

- Exact keyword 匹配 (use full-text search)
- 结构化 查询 (use SQL, filters)
- Small datasets where manual 标签ing is faster (<100 items)
- Tasks where explainability matters more than accuracy (嵌入s are opaque)

## 模型 selection

选择based on your constraints:

- **Need an API, best value**: OpenAI text-嵌入-3-small (1536d, $0.02/1M 词元)
- **Need maximum accuracy**: Voyage-3 (1024d, $0.06/1M 词元, highest MTEB)
- **Need local/private**: BGE-M3 (1024d, free, multilingual, GPU recommended)
- **Need fast local prototyping**: all-MiniLM-L6-v2 (384d, free, runs on CPU)
- **Need multilingual**: Cohere embed-v3 (1024d) or BGE-M3 (both strong multilingual)

Rule: never mix 嵌入 模型 between indexing and querying. Vectors from different 模型 live in incompatible spaces.

## 分块 rules

1. 目标 256-512 词元 per 分块 with 50-词元 overlap
2. Never split mid-sentence if you can avoid it
3. Include metadata (来源 file, section title, position) with every 分块
4. For 结构化 docs (Markdown, HTML), split at heading boundaries first
5. Test 分块 质量 by searching for known answers and checking 检索

## 相似度 指标 selection

- **Cosine 相似度**: default choice, handles variable-length 文本, normalized
- **Dot product**: use when vectors are already unit-normalized (OpenAI 模型 are), slightly faster
- **Euclidean distance**: use for clustering, when absolute position matters

All three give the same 排序 when vectors are normalized. The choice only matters for non-normalized vectors.

## Storage 优化

Three levels of 压缩, stackable:

1. **Matryoshka truncation**: reduce 维度 (1536 -> 256 = 6x savings, 3-5% accuracy 损失)
2. **Float16 量化**: halve storage per 维度 (2x savings, <1% accuracy 损失)
3. **Binary 量化**: 1 bit per 维度 (32x savings, 5-10% accuracy 损失, use with rescoring)

生产 pattern: binary search over full 语料库, rescore top-1000 with float32 vectors.

## Retrieve-then-rerank

Two-stage 流水线 for best accuracy:

1. Bi-encoder retrieves top-100 candidates (fast, uses pre-computed 嵌入s)
2. Cross-encoder reranks to top-10 (slow, processes each query-doc pair)

这beats single-stage 检索 by 10-15% on precision 指标. Use when accuracy matters more than 延迟.

## Common mistakes

- Using different 嵌入 模型 for indexing and querying
- 嵌入 entire 文档 instead of chunks (嵌入 becomes average of everything)
- Not normalizing vectors before cosine 相似度 (most 模型 pre-normalize, but verify)
- Ignoring 分块 overlap (sentences split at boundaries lose 上下文)
- Storing only vectors without the original 文本 (you need both for 检索)
- Not re-嵌入 when the 模型 changes (old vectors are incompatible)
- Choosing 维度 based on accuracy alone (storage and 延迟 规模 linearly with 维度)

## 调试 嵌入s

如果search results are poor:

1. Verify the 查询 嵌入 is non-zero (empty or whitespace 输入 produces zero vectors)
2. Check a known-relevant 文档's 相似度 分数 manually
3. Try rephrasing the 查询 to match 文档 词表
4. Inspect 分块 boundaries to ensure relevant content is not split across chunks
5. 比较top-k results across 指标 (cosine, dot, euclidean) to spot 归一化 issues
6. Test with a trivially 匹配 查询 (copy a sentence from a 文档) to confirm the 流水线 works

## 生产 参数

- 分块 size: 256-512 词元
- 分块 overlap: 50 词元 (10-20% of 分块 size)
- Top-k 检索: 5-10 for direct use, 50-100 for 重排
- 相似度 阈值: 0.7+ for cosine (below this, results are usually irrelevant)
- 批次 嵌入: process 100-500 texts per API call for throughput
- Index rebuild: re-embed when the 模型 changes or 文档 update significantly
