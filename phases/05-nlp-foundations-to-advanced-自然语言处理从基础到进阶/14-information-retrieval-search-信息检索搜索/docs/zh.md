# 信息检索与搜索

> 说明：BM25 is precise but brittle. Dense casts a wide net but misses keywords. Hybrid is the 2026 默认. Everything else is tuning.

**类型:** 构建
**语言:** Python
**先修要求:** 说明：Phase 5 · 02 (BoW + TF-IDF), Phase 5 · 04 (GloVe, FastText, Subword)
**时间:** 约75分钟

## 问题

这个 用户 types "what happens if someone lies to get money" 与 expects to find the statute 这 actually covers 这: "Section 420 IPC." A keyword search misses it entirely (no shared vocabulary). A 语义 search misses it if the 嵌入 were not trained on legal 文本. Real search has to handle both.

IR is the 流水线 under every RAG 系统, every search bar, every docs site's fuzzy lookup. The 2026 architecture 这 works in 生产 is not a single 方法. It is a chain of complementary methods, each catching the failures of the one before.

说明：本课 builds each piece 与 names 这 failures each catches.

## 概念

说明：![Hybrid 检索: BM25 + dense + RRF + cross-encoder rerank](../assets/retrieval.svg)

Four layers. Pick the ones you need.

1. 说明：**Sparse 检索 (BM25).** 快, precise on exact matches, terrible on semantics. Run over an inverted index. Sub-10ms per 查询 on millions of 文档. Gets you statute references, product codes, error messages, named entities right.
2. **Dense 检索.** Encode 查询 与 文档 到 向量. Nearest neighbor search. Captures paraphrases 与 语义 相似度. Misses exact keyword matches 这 differ by one character. 50-200ms per 查询 使用 FAISS 或 a 向量 DB.
3. **Fusion.** Merge the ranked lists 从 sparse 与 dense. Reciprocal Rank Fusion (RRF) is the easy 默认 因为 it ignores 原始 scores (这 live in different scales) 与 only uses rank positions. Weighted fusion is an option when you know one signal dominates 面向 your 领域.
4. 说明：**Cross-encoder rerank.** Take the top-30 从 fusion. Run a cross-encoder (查询 + 文档 together, scoring each pair). Keep the top-5. Cross-encoders are slower per pair than bi-encoders but far more 准确. You amortize by only running them on the top-30.

说明：Three-way 检索 (BM25 + dense + learned-sparse like SPLADE) outperforms two-way in 2026 benchmarks but needs infrastructure 面向 learned-sparse indexes. 面向 most teams, two-way plus cross-encoder rerank is the sweet spot.

## 动手构建

### Step 1: BM25 从 scratch

```python
import math
import re
from collections import Counter

TOKEN_RE = re.compile(r"[a-z0-9]+")


def tokenize(text):
    return TOKEN_RE.findall(text.lower())


class BM25:
    def __init__(self, corpus, k1=1.5, b=0.75):
        if not corpus:
            raise ValueError("corpus must not be empty")
        self.corpus = [tokenize(d) for d in corpus]
        self.k1 = k1
        self.b = b
        self.n_docs = len(self.corpus)
        self.avg_dl = sum(len(d) for d in self.corpus) / self.n_docs
        self.df = Counter()
        for doc in self.corpus:
            for term in set(doc):
                self.df[term] += 1

    def idf(self, term):
        n = self.df.get(term, 0)
        return math.log(1 + (self.n_docs - n + 0.5) / (n + 0.5))

    def score(self, query, doc_idx):
        q_tokens = tokenize(query)
        doc = self.corpus[doc_idx]
        dl = len(doc)
        freq = Counter(doc)
        score = 0.0
        for term in q_tokens:
            f = freq.get(term, 0)
            if f == 0:
                continue
            numerator = f * (self.k1 + 1)
            denominator = f + self.k1 * (1 - self.b + self.b * dl / self.avg_dl)
            score += self.idf(term) * numerator / denominator
        return score

    def rank(self, query, top_k=10):
        scored = [(self.score(query, i), i) for i in range(self.n_docs)]
        scored.sort(reverse=True)
        return scored[:top_k]
```

说明：Two parameters worth knowing. `k1=1.5` controls term-frequency saturation; higher means more weight on term repetition. `b=0.75` controls length normalization; 0 ignores 文档 length, 1 fully normalizes. The defaults are Robertson's recommendations 从 the original paper 与 rarely need tuning.

### Step 2: dense 检索 使用 a bi-encoder

```python
from sentence_transformers import SentenceTransformer
import numpy as np


def build_dense_index(corpus, model_id="sentence-transformers/all-MiniLM-L6-v2"):
    encoder = SentenceTransformer(model_id)
    embeddings = encoder.encode(corpus, normalize_embeddings=True)
    return encoder, embeddings


def dense_search(encoder, embeddings, query, top_k=10):
    q_emb = encoder.encode([query], normalize_embeddings=True)
    sims = (embeddings @ q_emb.T).flatten()
    order = np.argsort(-sims)[:top_k]
    return [(float(sims[i]), int(i)) for i in order]
```

L2-normalize 嵌入 so dot product equals cosine. `all-MiniLM-L6-v2` is 384-dim, 快, 与 strong enough 面向 most English 检索. 面向 多语言 work, use `paraphrase-multilingual-MiniLM-L12-v2`. 面向 top 准确率, `bge-large-en-v1.5` 或 `e5-large-v2`.

### Step 3: Reciprocal Rank Fusion

```python
def reciprocal_rank_fusion(rankings, k=60):
    scores = {}
    for ranking in rankings:
        for rank, (_, doc_idx) in enumerate(ranking):
            scores[doc_idx] = scores.get(doc_idx, 0.0) + 1.0 / (k + rank + 1)
    fused = sorted(scores.items(), key=lambda x: x[1], reverse=True)
    return [(score, doc_idx) for doc_idx, score in fused]
```

说明：这个 `k=60` constant comes 从 the original RRF paper. Higher `k` flattens the contribution of rank differences; lower `k` makes top ranks dominate. 60 is the published 默认 与 rarely needs tuning.

### Step 4: hybrid search + rerank

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")


def hybrid_search(query, bm25, encoder, dense_embeddings, corpus, top_k=5, pool_size=30, reranker=reranker):
    sparse_ranking = bm25.rank(query, top_k=pool_size)
    dense_ranking = dense_search(encoder, dense_embeddings, query, top_k=pool_size)
    fused = reciprocal_rank_fusion([sparse_ranking, dense_ranking])[:pool_size]

    pairs = [(query, corpus[doc_idx]) for _, doc_idx in fused]
    scores = reranker.predict(pairs)
    reranked = sorted(zip(scores, [doc_idx for _, doc_idx in fused]), reverse=True)
    return reranked[:top_k]
```

Three stages composed. BM25 finds lexical matches. Dense finds 语义 matches. RRF merges the two rankings 不使用 needing score calibration. Cross-encoder rescores the top-30 using 查询-文档 pairs together, 这 captures fine-grained relevance the bi-encoder missed. Keep top-5.

### Step 5: 评估

|Metric|Meaning|
|--------|---------|
|召回率@k|说明：Of queries 其中 the 正确 文档 exists, how often is it in the top-k?|
|MRR (Mean Reciprocal Rank)|Average of 1/rank of first relevant 文档.|
|nDCG@k|说明：Accounts 面向 relevance gradations, not just binary relevant/not.|

说明：对于 RAG specifically, **召回率@k** of the retriever is the most important number. Your reader cannot 答案 if the right passage is not in the retrieved set.

Debugging tip: 面向 failing queries, diff the sparse 与 dense rankings. If one finds the right 文档 与 the other does not, you have a vocabulary mismatch (fix: add the missing half) 或 a 语义 ambiguity (fix: better 嵌入 或 a reranker).

## 投入使用

这个 2026 stack:

|Scale|Stack|
|-------|-------|
|1k-100k docs|In-memory BM25 + `all-MiniLM-L6-v2` 嵌入 + RRF. No separate DB.|
|100k-10M docs|说明：FAISS 或 pgvector 面向 dense + Elasticsearch / OpenSearch 面向 BM25. Run in parallel.|
|10M+ docs|说明：Qdrant / Weaviate / Vespa / Milvus 使用 hybrid support. Cross-encoder rerank on top-30.|
|Best-质量 frontier|说明：Three-way (BM25 + dense + SPLADE) + ColBERT late-interaction reranking|

Whatever you pick, 预算 面向 评估. 基准 检索 召回率 before benchmarking end-to-end RAG 准确率. A reader cannot fix what the retriever missed.

### 这个 hard-won lessons 从 2026 生产 RAG

- **80% of RAG failures trace to ingestion 与 分块, not the 模型.** Teams spend weeks swapping LLMs 与 tuning prompts while the 检索 quietly returns the 错误 context every third 查询. Fix 分块 first.
- **Chunking strategy matters more than chunk size.** 固定-size splits break tables, code, 与 nested headers. 句子-aware is the 默认; 语义 或 LLM-based 分块 pays off 面向 technical docs 与 product manuals.
- **Parent-doc pattern.** Retrieve 小 "child" chunks 面向 精确率. When multiple children 从 the same parent section appear, swap in the parent block to preserve context. This consistently lifts 答案 质量 不使用 retraining.
- **k_rerank=3 is usually optimal.** Every extra chunk past 这 adds 词元 成本 与 生成 延迟 不使用 lifting 答案 质量. If k=8 is still better than k=3 面向 you, the reranker is underperforming.
- **HyDE / 查询 expansion.** Generate a hypothetical 答案 从 the 查询, embed 这, retrieve. Bridges the phrasing gap between 短 questions 与 长 文档. Free 精确率 lift 使用 no 训练.
- 说明：**Context 预算 under 8K 词元.** Consistent hits at 这 limit mean the reranker threshold is too loose.
- **Version everything.** Prompts, 分块 rules, 嵌入 模型, reranker. Any drift silently breaks 答案 质量. CI gates on faithfulness, context 精确率, 与 unanswered-问题 rate block regressions before users see them.
- 说明：**Three-way 检索 (BM25 + dense + learned-sparse like SPLADE) outperforms two-way** on 2026 benchmarks, especially 面向 queries mixing proper nouns 使用 semantics. Ship it when infrastructure supports SPLADE indexes.

说明：Proper 检索 design reduces hallucinations by 70-90% according to 2026 industry measurements. Most RAG performance gains come 从 better 检索, not 模型 fine-tuning.

## 交付成果

保存为 `outputs/skill-retrieval-picker.md`:

```markdown
---
name: retrieval-picker
description: Pick a retrieval stack for a given corpus and query pattern.
version: 1.0.0
phase: 5
lesson: 14
tags: [nlp, retrieval, rag, search]
---

Given requirements (corpus size, query pattern, latency budget, quality bar, infra constraints), output:

1. Stack. BM25 only, dense only, hybrid (BM25 + dense + RRF), hybrid + cross-encoder rerank, or three-way (BM25 + dense + learned-sparse).
2. Dense encoder. Name the specific model. Match to language(s), domain, and context length.
3. Reranker. Name the specific cross-encoder model if used. Flag that rerank adds 30-100ms latency on top-30.
4. Evaluation plan. Recall@10 is the primary retriever metric. MRR for multi-answer. Baseline first, incremental improvements measured against it.

Refuse to recommend dense-only for corpora with named entities, error codes, or product SKUs unless the user has evidence dense handles exact matches. Refuse to skip reranking for high-stakes retrieval (legal, medical) where the final top-5 decides the user's answer.
```

## 练习

1. **Easy.** Implement `hybrid_search` above on a 500-文档 语料库. Test 20 queries. Compare 召回率 at 5 between BM25-only, dense-only, 与 hybrid.
2. **Medium.** Add MRR calculation. 面向 each test 查询 使用 a known 正确 文档, find the rank of the 正确 doc in BM25, dense, 与 hybrid rankings. Report the MRR 面向 each.
3. **Hard.** Fine-tune a dense encoder on your 领域 using MultipleNegativesRankingLoss (句子 Transformers). Build a 训练 set 从 500 查询-文档 pairs. Compare pre- 与 post-fine-tune 召回率.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|BM25|Keyword search|说明：Okapi BM25. Scores 文档 by term frequency, IDF, 与 length.|
|Dense 检索|Vector search|Encode 查询 + doc 到 向量, find nearest neighbors.|
|Bi-encoder|嵌入 模型|Encodes 查询 与 doc independently. 快 at 查询 time.|
|Cross-encoder|Reranker 模型|Encodes 查询 + doc together. 慢 but 准确.|
|RRF|Rank fusion|Combine two rankings by summing `1/(k + rank)`.|
|召回率@k|Retrieval 指标|说明：Fraction of queries 其中 a relevant doc is in the top-k.|

## 延伸阅读

- 说明：[说明：Robertson 与 Zaragoza (2009). The Probabilistic Relevance Framework: BM25 与 Beyond](https://www.staff.city.ac.uk/~sbrp622/papers/foundations_bm25_review.pdf)，the definitive BM25 treatment.
- [说明：Karpukhin et al. (2020). Dense Passage Retrieval 面向 开放-领域 QA](https://arxiv.org/abs/2004.04906)，DPR, the canonical bi-encoder.
- [说明：Formal et al. (2021). SPLADE: Sparse Lexical 与 Expansion 模型](https://arxiv.org/abs/2107.05720)，the learned-sparse retriever 这 closes the gap 使用 dense.
- 说明：[说明：Cormack, Clarke, Büttcher (2009). Reciprocal Rank Fusion outperforms Condorcet 与 individual Rank Learning Methods](https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf)，RRF paper.
- 说明：[说明：Khattab 与 Zaharia (2020). ColBERT: Efficient 与 Effective Passage Search](https://arxiv.org/abs/2004.12832)，late-interaction 检索.
