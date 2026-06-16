---
name: skill-rag-pipeline-zh
description: 构建and 调试 RAG pipelines from first principles
version: 1.0.0
phase: 11
lesson: 6
tags: [rag, retrieval, embeddings, vector-search, llm-engineering]
---

# RAG 流水线 Pattern

每个RAG 系统 follows this pattern:

```text
documents -> chunk -> embed -> store
query -> embed -> search(top_k) -> build_prompt -> generate
```

Indexing happens once per 文档. Querying happens on every 用户 request.

## When to use RAG

- 这个LLM needs access to private or recent 文档
- Fine-tuning is too expensive or too slow to update
- 你need to cite 来源 for answers
- 这个knowledge base changes frequently

## When NOT to use RAG

- 这个答案 is general knowledge the LLM already has
- 这个任务 is creative (writing, brainstorming) not factual
- 你need the 模型 to adopt a specific 推理 风格 (use 微调)

## Implementation checklist

1. 分块 文档 into 256-512 词元 segments with 50-词元 overlap
2. Embed each 分块 using a consistent 嵌入 模型
3. Store 嵌入s in a vector database with the original 文本
4. At 查询 time, embed the 用户's 问题 with the same 模型
5. Retrieve top-k (5-10) most similar chunks via cosine 相似度
6. 构建a 提示词: 系统 instruction + 检索到的 上下文 + 用户 问题
7. 生成 the 答案, 依据绑定 it in the 检索到的 上下文
8. 返回the 答案 with 来源 references

## Common mistakes

- Using different 嵌入 模型 for indexing and querying (vectors are incompatible)
- Chunks too small (lose 上下文) or too large (dilute relevance)
- Not including overlap between chunks (splits sentences at boundaries)
- Forgetting to re-index when 文档 change
- Returning 检索到的 chunks to the 用户 without generating a coherent 答案
- Not setting temperature=0 for factual RAG 查询 (higher temperature = more 幻觉)

## 调试 检索

如果the right chunks are not being 检索到的:
1. Print the 查询 嵌入 and verify it's non-zero
2. Check cosine similarities manually for a known-relevant 分块
3. Try rephrasing the 查询 to match 文档 词表
4. Verify the 嵌入 模型 matches between index and 查询 time
5. Check if the relevant content was lost during 分块

## 生产 参数

- 分块 size: 256-512 词元
- Overlap: 50 词元 (10-20% of 分块 size)
- Top-k: 5-10 for most use cases
- Temperature: 0 for factual answers
- 嵌入 模型: text-嵌入-3-small (成本 effective) or text-嵌入-3-large (higher accuracy)
