# 面向 RAG 的分块策略

> Chunking 配置 influences 检索 质量 as much as the choice of 嵌入 模型 (Vectara NAACL 2025). Get 分块 错误 与 no amount of reranking saves you.

**类型:** 构建
**语言:** Python
**先修要求:** 说明：Phase 5 · 14 (Information Retrieval), Phase 5 · 22 (嵌入 Models)
**时间:** 约60分钟

## 问题

You put a 50-page contract 到 a RAG 系统. 用户 asks: "What is the termination clause?" The retriever returns the cover page. Why? 因为 the 模型 was trained on 512-词元 chunks 与 the termination clause sits 20 pages in, split across a page break, 使用 no local keywords tying it to the 查询.

这个 fix is not "buy a better 嵌入 模型." The fix is 分块. How big? Overlap? 其中 to split? 使用 surrounding context?

Feb 2026 benchmarks show surprising results:

- Vectara's 2026 study: recursive 512-词元 分块 beat 语义 分块 69% → 54% 准确率.
- 说明：SPLADE + Mistral-8B on Natural Questions: overlap provided zero measurable benefit.
- 说明：Context cliff: response 质量 drops sharply around 2,500 词元 of context.

这个 "obvious" 答案 (语义 分块, 20% overlap, 1000 词元) is often 错误. This lesson builds intuition 面向 six strategies 与 tells you when to reach 面向 这.

## 概念

说明：![Six 分块 strategies visualized on one passage](../assets/chunking.svg)

**固定 分块.** Split every N characters 或 词元. Simplest 基线. Breaks mid-句子. Good compression, bad coherence.

说明：**Recursive.** LangChain's `RecursiveCharacterTextSplitter`. Try splitting on `\n\n` first, then `\n`, then `.`, then space. Falls back cleanly. The 2026 默认.

**语义.** Embed each 句子. 计算 cosine 相似度 between adjacent sentences. Split 其中 相似度 drops below a threshold. Preserves 主题 coherence. Slower; sometimes produces tiny 40-词元 fragments 这 hurt 检索.

**句子.** Split on 句子 boundaries. One 句子 per chunk 或 a window of N sentences. Matches 语义 分块 up to ~5k 词元 at a fraction of the 成本.

**Parent-文档.** Store 小 child chunks 面向 检索 *与* the larger parent chunk 面向 context. Retrieve by child; return parent. Degrades gracefully: bad child chunks still return reasonable parents.

**Late 分块 (2024).** Embed the whole 文档 at the 词元 level first, then pool 词元 嵌入 到 chunk 嵌入. Preserves cross-chunk context. Works 使用 长-context embedders (BGE-M3, Jina v3). Higher 计算.

**Contextual 检索 (Anthropic, 2024).** Prepend each chunk 使用 an LLM-generated 摘要 of its position in the 文档 ("This chunk is section 3.2 of the termination clauses..."). 35-50% 检索 improvement in Anthropic's own 基准. Expensive to index.

### 这个 rule 这 beats every 默认

Match the chunk size to the 查询 type:

|Query type|Chunk size|
|------------|-----------|
|Factoid ("what is the CEO's name?")|256-512 词元|
|Analytical / multi-hop|512-1024 词元|
|Whole-section comprehension|1024-2048 词元|

NVIDIA's 2026 基准. The chunk should be big enough to contain the 答案 plus local context, 小 enough 这 the retriever's top-K returns focus on the 答案 rather than context 噪声.

## 动手构建

### Step 1: 固定 与 recursive 分块

```python
def chunk_fixed(text, size=512, overlap=0):
    step = size - overlap
    return [text[i:i + size] for i in range(0, len(text), step)]


def chunk_recursive(text, size=512, seps=("\n\n", "\n", ". ", " ")):
    if len(text) <= size:
        return [text]
    for sep in seps:
        if sep not in text:
            continue
        parts = text.split(sep)
        chunks = []
        buf = ""
        for p in parts:
            if len(p) > size:
                if buf:
                    chunks.append(buf)
                    buf = ""
                chunks.extend(chunk_recursive(p, size=size, seps=seps[1:] or (" ",)))
                continue
            candidate = buf + sep + p if buf else p
            if len(candidate) <= size:
                buf = candidate
            else:
                if buf:
                    chunks.append(buf)
                buf = p
        if buf:
            chunks.append(buf)
        return [c for c in chunks if c.strip()]
    return chunk_fixed(text, size)
```

### Step 2: 语义 分块

```python
def chunk_semantic(text, encoder, threshold=0.6, min_chars=200, max_chars=2048):
    sentences = split_sentences(text)
    if not sentences:
        return []
    embs = encoder.encode(sentences, normalize_embeddings=True)
    chunks = [[sentences[0]]]
    for i in range(1, len(sentences)):
        sim = float(embs[i] @ embs[i - 1])
        current_len = sum(len(s) for s in chunks[-1])
        if sim < threshold and current_len >= min_chars:
            chunks.append([sentences[i]])
        else:
            chunks[-1].append(sentences[i])

    result = []
    for group in chunks:
        text_group = " ".join(group)
        if len(text_group) > max_chars:
            result.extend(chunk_recursive(text_group, size=max_chars))
        else:
            result.append(text_group)
    return result
```

说明：Tune `threshold` on your 领域. Too high → fragments. Too low → one giant chunk.

### Step 3: parent-文档

```python
def chunk_parent_child(text, parent_size=2048, child_size=256):
    parents = chunk_recursive(text, size=parent_size)
    mapping = []
    for p_idx, parent in enumerate(parents):
        children = chunk_recursive(parent, size=child_size)
        for child in children:
            mapping.append({"child": child, "parent_idx": p_idx, "parent": parent})
    return mapping


def retrieve_parent(child_query, mapping, encoder, top_k=3):
    child_embs = encoder.encode([m["child"] for m in mapping], normalize_embeddings=True)
    q_emb = encoder.encode([child_query], normalize_embeddings=True)[0]
    scores = child_embs @ q_emb
    top = np.argsort(-scores)[:top_k]
    seen, parents = set(), []
    for i in top:
        if mapping[i]["parent_idx"] not in seen:
            parents.append(mapping[i]["parent"])
            seen.add(mapping[i]["parent_idx"])
    return parents
```

说明：关键 insight: dedupe parents. Multiple children can map to the same parent; returning all would waste context.

### Step 4: contextual 检索 (Anthropic pattern)

```python
def contextualize_chunks(document, chunks, llm):
    context_prompts = [
        f"""<document>{document}</document>
Here is the chunk to situate: <chunk>{c}</chunk>
Write 50-100 words placing this chunk in the document's context."""
        for c in chunks
    ]
    contexts = llm.batch(context_prompts)
    return [f"{ctx}\n\n{c}" for ctx, c in zip(contexts, chunks)]
```

说明：Index the contextualized chunks. At 查询 time, 检索 benefits 从 the extra surrounding signal.

### Step 5: evaluate

```python
def recall_at_k(queries, corpus_chunks, encoder, k=5):
    chunk_embs = encoder.encode(corpus_chunks, normalize_embeddings=True)
    hits = 0
    for q_text, gold_idxs in queries:
        q_emb = encoder.encode([q_text], normalize_embeddings=True)[0]
        top = np.argsort(-(chunk_embs @ q_emb))[:k]
        if any(i in gold_idxs for i in top):
            hits += 1
    return hits / len(queries)
```

说明：Always 基准. The "best" strategy 面向 your 语料库 may not match any blog post.

## 陷阱

- 说明：**Chunking evaluated only on factoid queries.** Multi-hop queries reveal very different winners. Use a 查询-type-stratified eval set.
- **语义 分块 不使用 a minimum size.** Produces 40-词元 fragments 这 hurt 检索. Always enforce `min_tokens`.
- 说明：**Overlap as cargo cult.** 2026 studies find overlap often provides zero benefit 与 doubles index 成本. Measure, do not assume.
- 说明：**No min/max enforcement.** Chunks of 5 词元 或 5000 词元 both break 检索. Clamp.
- 说明：**Cross-doc 分块.** Never let a chunk span two 文档. Always chunk per-doc, then merge.

## 投入使用

这个 2026 stack:

|Situation|Strategy|
|-----------|----------|
|First build, unknown 语料库|Recursive, 512 词元, no overlap|
|Factoid QA|Recursive, 256-512 词元|
|Analytical / multi-hop|Recursive, 512-1024 词元 + parent-文档|
|Heavy cross-reference (contracts, papers)|Late 分块 或 contextual 检索|
|Conversational / dialog 语料库|Turn-level chunks + speaker metadata|
|短 utterances (tweets, reviews)|One 文档 = one chunk|

Start 使用 recursive 512. Measure 召回率@5 on a 50-查询 eval set. Tune 从 there.

## 交付成果

保存为 `outputs/skill-chunker.md`:

```markdown
---
name: chunker
description: Pick a chunking strategy, size, and overlap for a given corpus and query distribution.
version: 1.0.0
phase: 5
lesson: 23
tags: [nlp, rag, chunking]
---

Given a corpus (document types, avg length, domain) and query distribution (factoid / analytical / multi-hop), output:

1. Strategy. Recursive / sentence / semantic / parent-document / late / contextual. Reason.
2. Chunk size. Token count. Reason tied to query type.
3. Overlap. Default 0; justify if >0.
4. Min/max enforcement. `min_tokens`, `max_tokens` guards.
5. Evaluation plan. Recall@5 on 50-query stratified eval set (factoid, analytical, multi-hop).

Refuse any chunking strategy without min/max chunk size enforcement. Refuse overlap above 20% without an ablation showing it helps. Flag semantic chunking recommendations without a min-token floor.
```

## 练习

1. **Easy.** Chunk one 20-page 文档 使用 固定(512, 0), recursive(512, 0), 与 recursive(512, 100). Compare chunk counts 与 boundary 质量.
2. **Medium.** Build a 30-查询 eval set over 5 文档. Measure 召回率@5 面向 recursive, 语义, 与 parent-文档. 这 wins? Does it match the blog posts?
3. **Hard.** Implement contextual 检索. Measure MRR improvement over 基线 recursive. Report index 成本 (LLM calls) vs 准确率 gain.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|Chunk|一个 piece of a doc|Sub-文档 unit 这 gets embedded, indexed, 与 retrieved.|
|Overlap|Safety margin|说明：N 词元 shared between adjacent chunks; often useless in 2026 benchmarks.|
|语义 分块|Smart 分块|Split 其中 adjacent-句子 嵌入 相似度 drops.|
|Parent-文档|Two-level 检索|Retrieve 小 children, return larger parents.|
|Late 分块|Chunk after 嵌入|Embed full doc at 词元 level, pool 到 chunk 向量.|
|Contextual 检索|Anthropic's trick|说明：LLM-generated 摘要 prepended to each chunk before indexing.|
|Context cliff|2500-词元 wall|质量 drop observed around 2.5k context 词元 in RAG (Jan 2026).|

## 延伸阅读

- 说明：[说明：Yepes et al. / LangChain，Recursive Character Splitting docs](https://python.langchain.com/docs/how_to/recursive_text_splitter/)，the 默认 in 生产.
- 说明：[说明：Vectara (2024, NAACL 2025). Chunking configurations analysis](https://arxiv.org/abs/2410.13070)，分块 matters as much as 嵌入 choice.
- 说明：[Jina AI，Late Chunking in 长-Context 嵌入 Models (2024)](https://jina.ai/news/late-chunking-in-long-context-embedding-models/)，the late 分块 paper.
- 说明：[Anthropic，Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval)，35-50% 检索 improvement 使用 LLM-generated context prefixes.
- 说明：[NVIDIA 2026 chunk-size 基准，Premai 摘要](https://blog.premai.io/rag-chunking-strategies-the-2026-benchmark-guide/)，chunk size by 查询 type.
