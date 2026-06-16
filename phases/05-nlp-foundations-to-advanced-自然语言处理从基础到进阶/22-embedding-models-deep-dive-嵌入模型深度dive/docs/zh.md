# 嵌入模型，2026 深入解析

> Word2Vec gave you a 向量 per 词. 现代 嵌入 models give you a 向量 per passage, cross-lingual, 使用 sparse, dense, 与 multi-向量 views, sized to fit your index. Pick 错误 与 your RAG retrieves the 错误 thing.

**类型:** 学习
**语言:** Python
**先修要求:** 说明：Phase 5 · 03 (Word2Vec), Phase 5 · 14 (Information Retrieval)
**时间:** 约60分钟

## 问题

Your RAG 系统 retrieves the 错误 passage 40% of the time. The culprit is rarely the 向量 database 或 the prompt. It is the 嵌入 模型.

说明：Choosing an 嵌入 in 2026 means picking across five axes:

1. **Dense vs sparse vs multi-向量.** One 向量 per passage, 或 one per 词元, 或 a sparse weighted bag of 词.
2. 说明：**Language coverage.** Monolingual English models still win on English-only tasks. Multilingual models win when corpora are mixed.
3. 说明：**Context length.** 512 词元 vs 8,192 vs 32,768，与 real effective capacity is often 60-70% of the advertised max.
4. **Dimension 预算.** 3,072 floats at full 精确率 = 12 KB per 向量. At 100M 向量, 存储 is $1,300/month. Matryoshka truncation cuts this 4×.
5. **开放 vs hosted.** 开放-weight means you control the stack 与 数据. Hosted means you trade control 面向 always-最新.

说明：本课 names the tradeoffs so you can pick on evidence, not on whatever was popular last quarter.

## 概念

![Dense, sparse, 与 multi-向量 嵌入](../assets/embedding-modes.svg)

**Dense 嵌入.** One 向量 per passage (usually 384-3,072 dimensions). Cosine 相似度 ranks passages by 语义 proximity. OpenAI `text-embedding-3-large`, BGE-M3 dense mode, Voyage-3. 默认 choice.

**Sparse 嵌入.** SPLADE-style. A transformer predicts a weight 面向 every vocab 词元, then zeros out most of them. Result is a sparse 向量 of size |vocab|. Captures lexical matching (like BM25) but 使用 learned term weights. Strong on keyword-heavy queries.

**Multi-向量 (late interaction).** ColBERTv2, Jina-ColBERT. One 向量 per 词元. Scoring 使用 MaxSim: 面向 each 查询 词元, find the most similar 文档 词元, sum the scores. More expensive to store 与 score, but wins on 长 queries 与 领域-specific corpora.

**BGE-M3: all three at once.** Single 模型 outputs dense, sparse, 与 multi-向量 representations simultaneously. Each can be queried independently; scores fuse via weighted sum. The 2026 默认 when you want flexibility 从 one checkpoint.

**Matryoshka Representation Learning.** Trained so the first N dimensions of the 向量 form a useful standalone 嵌入. Truncate a 1,536-dim 向量 to 256 dim 与 pay ~1% 准确率 面向 6× 存储 savings. Supported by OpenAI 文本-3, Cohere v4, Voyage-4, Jina v5, Gemini 嵌入 2, Nomic v1.5+.

### 这个 MTEB leaderboard tells a partial story

Massive 文本 嵌入 基准，56 tasks across 8 任务 types at launch (2022), expanded to 100+ tasks in MTEB v2. In early 2026, Gemini 嵌入 2 tops 检索 (67.71 MTEB-R). Cohere embed-v4 leads general (65.2 MTEB). BGE-M3 leads 开放-weight 多语言 (63.0). The leaderboard is necessary but not sufficient，always 基准 on your 领域.

### 这个 three-tier pattern

|Use case|Pattern|
|----------|---------|
|快 first-pass|Dense bi-encoder (BGE-M3, 文本-3-小)|
|召回率 boost|Sparse (SPLADE, BGE-M3 sparse) + RRF fuse|
|精确率 on top-50|Multi-向量 (ColBERTv2) 或 cross-encoder reranker|

大多数 生产 stacks use all three.

## 动手构建

### Step 1: 基线，dense 嵌入 使用 句子-BERT

```python
from sentence_transformers import SentenceTransformer
import numpy as np

encoder = SentenceTransformer("BAAI/bge-small-en-v1.5")
corpus = [
    "The first iPhone launched in 2007.",
    "Apple released the iPod in 2001.",
    "Android is an operating system from Google.",
]
emb = encoder.encode(corpus, normalize_embeddings=True)

query = "When was the iPhone released?"
q_emb = encoder.encode([query], normalize_embeddings=True)[0]
scores = emb @ q_emb
print(sorted(enumerate(scores), key=lambda x: -x[1]))
```

说明：`normalize_embeddings=True` makes the dot product equal cosine 相似度. Always set it.

### Step 2: Matryoshka truncation

```python
def truncate(vectors, dim):
    out = vectors[:, :dim]
    return out / np.linalg.norm(out, axis=1, keepdims=True)

emb_256 = truncate(emb, 256)
emb_128 = truncate(emb, 128)
```

说明：Re-normalize after truncation. Nomic v1.5, OpenAI 文本-3, 与 Voyage-4 are trained so this is lossless 面向 the first few levels. Non-Matryoshka models (original 句子-BERT) degrade sharply when truncated.

### Step 3: BGE-M3 multi-functionality

```python
from FlagEmbedding import BGEM3FlagModel

model = BGEM3FlagModel("BAAI/bge-m3", use_fp16=True)

output = model.encode(
    corpus,
    return_dense=True,
    return_sparse=True,
    return_colbert_vecs=True,
)
# output["dense_vecs"]:    (n_docs, 1024)
# output["lexical_weights"]: list of dict {token_id: weight}
# output["colbert_vecs"]:  list of (n_tokens, 1024) arrays
```

Three indexes, one 推理 call. Score fusion:

```python
dense_score = ... # cosine over dense_vecs
sparse_score = model.compute_lexical_matching_score(q_lex, d_lex)
colbert_score = model.colbert_score(q_col, d_col)
final = 0.4 * dense_score + 0.2 * sparse_score + 0.4 * colbert_score
```

Tune the weights on your 领域.

### Step 4: MTEB eval on a custom 任务

```python
from mteb import MTEB

tasks = ["ArguAna", "SciFact", "NFCorpus"]
evaluation = MTEB(tasks=tasks)
results = evaluation.run(encoder, output_folder="./mteb-results")
```

说明：Run your candidate models on a *representative* subset. Do not trust leaderboard rank alone，your 领域 matters.

### Step 5: hand-rolled cosine 从 scratch

See `code/main.py`. Averaged Hashing Trick 嵌入 (stdlib-only). Not competitive 使用 transformer 嵌入, but shows the shape: tokenize → 向量 → normalize → dot product.

## 陷阱

- **Same 模型 面向 查询 与 doc.** Some models (Voyage, Jina-ColBERT) use asymmetric encoding，查询 与 文档 pass through different paths. Always check the 模型 card.
- 说明：**Missing prefix.** `bge-*` models need `"Represent this sentence for searching relevant passages: "` prepended to queries. 3-5 point 召回率 gap if you forget.
- 说明：**Over-trimming Matryoshka.** 1,536 → 256 is usually safe. 1,536 → 64 is not. Validate on your eval set.
- 说明：**Context truncation.** Most models silently truncate inputs over their max length. 长 docs need 分块 (see lesson 23).
- **Ignoring 延迟 tail.** MTEB scores hide p99 延迟. A 600M 模型 might beat a 335M 模型 by 2 points but 成本 3× more per 查询.

## 投入使用

这个 2026 stack:

|Situation|Pick|
|-----------|------|
|English-only, 快, API|`text-embedding-3-large` 或 `voyage-3-large`|
|开放-weight, English|`BAAI/bge-large-en-v1.5`|
|开放-weight, 多语言|`BAAI/bge-m3` 或 `Qwen3-Embedding-8B`|
|长 context (32k+)|Voyage-3-大, Cohere embed-v4, Qwen3-嵌入-8B|
|CPU-only deployment|Nomic Embed v2 (137M params, MoE)|
|存储-constrained|Matryoshka-truncated + int8 quantization|
|Keyword-heavy queries|Add SPLADE sparse, RRF-fuse 使用 dense|

2026 pattern: start 使用 BGE-M3 或 文本-3-大, evaluate on your 领域 使用 MTEB, swap if a 领域-specific 模型 wins by more than 3 points.

## 交付成果

保存为 `outputs/skill-embedding-picker.md`:

```markdown
---
name: embedding-picker
description: Pick embedding model, dimension, and retrieval mode for a given corpus and deployment.
version: 1.0.0
phase: 5
lesson: 22
tags: [nlp, embeddings, retrieval]
---

Given a corpus (size, languages, domain, avg length), deployment target (cloud / edge / on-prem), latency budget, and storage budget, output:

1. Model. Named checkpoint or API. One-sentence reason.
2. Dimension. Full / Matryoshka-truncated / int8-quantized. Reason tied to storage budget.
3. Mode. Dense / sparse / multi-vector / hybrid. Reason.
4. Query prefix / template if required by the model card.
5. Evaluation plan. MTEB tasks relevant to domain + held-out domain eval with nDCG@10.

Refuse recommendations that truncate Matryoshka to <64 dims without domain validation. Refuse ColBERTv2 for corpora under 10k passages (overhead not justified). Flag long-document corpora (>8k tokens) routed to models with 512-token windows.
```

## 练习

1. 说明：**Easy.** Encode 100 sentences 使用 `bge-small-en-v1.5` at full dim (384), then at Matryoshka 128. Measure MRR drop on 10 queries.
2. **Medium.** Compare BGE-M3 dense, sparse, 与 colbert on 500 passages 从 your 领域. 这 wins on 召回率@10? Does RRF fusion beat the best single mode?
3. 说明：**Hard.** Run MTEB on three candidate models across your top-2 领域 tasks. Report MTEB score, p99 延迟 on a 100-查询 batch, 与 $/1M queries. Pick the Pareto-optimal one.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|Dense 嵌入|这个 向量|One 固定-size 向量 per 文本. Cosine 相似度 面向 ranking.|
|Sparse 嵌入|Learned BM25|说明：One weight per vocab 词元; mostly zeros; trained end-to-end.|
|Multi-向量|ColBERT-style|说明：One 向量 per 词元; MaxSim scoring; bigger index, better 召回率.|
|Matryoshka|Russian doll trick|First N dims are a 有效 smaller 嵌入 on their own.|
|MTEB|这个 基准|Massive 文本 嵌入 基准，56 tasks at launch, 100+ in v2.|
|BEIR|这个 检索 基准|说明：18 zero-shot 检索 tasks; often cited 面向 cross-领域 robustness.|
|Asymmetric encoding|Query ≠ doc path|模型 uses different projections 面向 queries 与 文档.|

## 延伸阅读

- 说明：[Reimers, Gurevych (2019). 句子-BERT](https://arxiv.org/abs/1908.10084)，the bi-encoder paper.
- 说明：[Muennighoff et al. (2022). MTEB: Massive 文本 嵌入 基准](https://arxiv.org/abs/2210.07316)，the leaderboard paper.
- 说明：[说明：Chen et al. (2024). BGE-M3: Multi-lingual, Multi-functionality, Multi-granularity](https://arxiv.org/abs/2402.03216)，the unified three-mode 模型.
- 说明：[说明：Kusupati et al. (2022). Matryoshka Representation Learning](https://arxiv.org/abs/2205.13147)，the dimension-ladder 训练 objective.
- 说明：[说明：Santhanam et al. (2022). ColBERTv2: Effective 与 Efficient Retrieval via Lightweight Late Interaction](https://arxiv.org/abs/2112.01488)，late interaction in 生产.
- 说明：[MTEB leaderboard on Hugging Face](https://huggingface.co/spaces/mteb/leaderboard)，live rankings.
