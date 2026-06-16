# 主题建模，LDA 与 BERTopic

> LDA: 文档 are mixtures of 主题, 主题 are distributions over 词. BERTopic: 文档 聚类 in 嵌入 space, clusters are 主题. Same goal, different decompositions.

**类型:** 学习
**语言:** Python
**先修要求:** Phase 5 · 02 (BoW + TF-IDF), Phase 5 · 03 (Word2Vec)
**时间:** 约45分钟

## 问题

说明：你有 10,000 customer support tickets, 50,000 news articles, 或 200,000 tweets. You need to know what the collection is about 不使用 reading it. You do not have labeled categories. You do not even know how many categories exist.

主题 modeling answers 这 不使用 supervision. Give it a 语料库, get back a 小 set of coherent 主题 与, 面向 each 文档, a 分布 over those 主题.

Two algorithmic families dominate. LDA (2003) treats each 文档 as a mixture of latent 主题 与 each 主题 as a 分布 over 词. 推理 is Bayesian. It still ships in 生产 其中 you need mixed-membership 主题 assignments 与 explainable 词-level 概率 distributions.

BERTopic (2020) encodes 文档 使用 BERT, reduces dimensionality 使用 UMAP, clusters 使用 HDBSCAN, 与 extracts 主题 词 via class-based TF-IDF. It wins on 短 文本, social media, 与 anything 其中 语义 相似度 matters more than 词 overlap. One 文档 gets one 主题, 这 is a limitation 面向 长-form content.

本课 builds intuition 面向 both 与 names 这 one to pick 面向 a given 语料库.

## 概念

说明：![LDA mixture 模型 vs BERTopic clustering](../assets/topic-modeling.svg)

**LDA generative story.** Each 主题 is a 分布 over 词. Each 文档 is a mixture of 主题. To generate a 词 in a 文档, sample a 主题 从 the 文档's mixture, then sample a 词 从 这 主题's 分布. 推理 reverses this: given observed 词, infer the 主题 分布 per 文档 与 the 词 分布 per 主题. Collapsed Gibbs sampling 或 variational Bayes does the math.

关键 LDA 输出:

- `doc_topic`: 矩阵 `(n_docs, n_topics)`, each row sums to 1 (文档's 主题 mixture).
- `topic_word`: 矩阵 `(n_topics, vocab_size)`, each row sums to 1 (主题's 词 分布).

**BERTopic 流水线.**

1. Encode each 文档 使用 a 句子 transformer (e.g., `all-MiniLM-L6-v2`). 384-dim 向量.
2. 说明：Reduce dimensionality 使用 UMAP to ~5 dimensions. BERT 嵌入 are too high-dim 面向 clustering.
3. 聚类 使用 HDBSCAN. Density-based, produces variable-size clusters 与 an "离群点" 标签.
4. 对于 each 聚类, 计算 class-based TF-IDF over the 聚类's 文档 to extract top 词.

输出 is one 主题 per 文档 (plus a -1 离群点 标签). Optionally, a soft membership via HDBSCAN's 概率 向量.

## 动手构建

### Step 1: LDA via scikit-learn

```python
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.decomposition import LatentDirichletAllocation
import numpy as np


def fit_lda(documents, n_topics=5, max_features=1000):
    cv = CountVectorizer(
        max_features=max_features,
        stop_words="english",
        min_df=2,
        max_df=0.9,
    )
    X = cv.fit_transform(documents)
    lda = LatentDirichletAllocation(
        n_components=n_topics,
        random_state=42,
        max_iter=50,
        learning_method="online",
    )
    doc_topic = lda.fit_transform(X)
    feature_names = cv.get_feature_names_out()
    return lda, cv, doc_topic, feature_names


def print_top_words(lda, feature_names, n_top=10):
    for idx, topic in enumerate(lda.components_):
        top_idx = np.argsort(-topic)[:n_top]
        words = [feature_names[i] for i in top_idx]
        print(f"topic {idx}: {' '.join(words)}")
```

说明：Notice: stopwords removed, min_df 与 max_df filter rare 与 ubiquitous terms, CountVectorizer (not TfidfVectorizer) 因为 LDA expects 原始 counts.

### Step 2: BERTopic (生产)

```python
from bertopic import BERTopic

topic_model = BERTopic(
    embedding_model="sentence-transformers/all-MiniLM-L6-v2",
    min_topic_size=15,
    verbose=True,
)

topics, probs = topic_model.fit_transform(documents)
info = topic_model.get_topic_info()
print(info.head(20))
valid_topics = info[info["Topic"] != -1]["Topic"].tolist()
for topic_id in valid_topics[:5]:
    print(f"topic {topic_id}: {topic_model.get_topic(topic_id)[:10]}")
```

这个 filter on `Topic != -1` drops BERTopic's 离群点 bucket (文档 HDBSCAN could not 聚类). `min_topic_size` controls HDBSCAN's minimum 聚类 size; BERTopic's 库 默认 is 10. This example sets it to 15 explicitly 面向 the lesson's scale. 面向 corpora over 10,000 文档, increase to 50 或 100.

### Step 3: 评估

两者都 methods 输出 主题 词. The 问题 is whether those 词 cohere.

- **主题 coherence (c_v).** Combines NPMI (normalized pointwise mutual information) of top-词 pairs over sliding-window contexts, aggregates the scores 到 主题 向量, 与 compares those 向量 via cosine 相似度. Higher is better. Use `gensim.models.CoherenceModel` 使用 `coherence="c_v"`.
- **主题 diversity.** Fraction of unique 词 across all 主题' top 词. Higher is better (主题 do not overlap).
- 说明：**Qualitative inspection.** Read the top 词 of each 主题. Do they name a real thing? Human judgment is still the last line of defense.

## 如何选择

|Situation|Pick|
|-----------|------|
|短 文本 (tweets, reviews, headlines)|BERTopic|
|长 文档 使用 主题 mixtures|LDA|
|No GPU / limited 计算|LDA 或 NMF|
|Need 文档-level multi-主题 distributions|LDA|
|LLM integration 面向 主题 labeling|BERTopic (direct support)|
|Resource-constrained 边缘 deployment|LDA|
|Max 语义 coherence|BERTopic|

这个 biggest practical consideration is 文档 length. BERT 嵌入 truncate; LDA counts work on whatever length. 面向 文档 longer than the 嵌入 模型's context, either chunk + aggregate 或 use LDA.

## 投入使用

这个 2026 stack:

- **BERTopic.** 默认 面向 短 文本 与 anything 其中 semantics matter.
- **`gensim.models.LdaModel`.** 经典 LDA 面向 生产, mature, battle-tested.
- **`sklearn.decomposition.LatentDirichletAllocation`.** Easy LDA 面向 experiments.
- **NMF.** Non-negative 矩阵 factorization. 快 alternative to LDA, comparable 质量 on 短 文本.
- 说明：**Top2Vec.** Similar design to BERTopic. Smaller community but good on some benchmarks.
- 说明：**FASTopic.** Newer, faster than BERTopic on very 大 corpora.
- 说明：**LLM-based labeling.** Run any clustering, then prompt a 模型 to name each 聚类.

## 交付成果

保存为 `outputs/skill-topic-picker.md`:

```markdown
---
name: topic-picker
description: Pick LDA or BERTopic for a corpus. Specify library, knobs, evaluation.
version: 1.0.0
phase: 5
lesson: 15
tags: [nlp, topic-modeling]
---

Given a corpus description (document count, avg length, domain, language, compute budget), output:

1. Algorithm. LDA / NMF / BERTopic / Top2Vec / FASTopic. One-sentence reason.
2. Configuration. Number of topics: `recommended = max(5, round(sqrt(n_docs)))`, clamped to 200 for corpora under 40,000 docs; permit >200 only when the corpus is genuinely large (>40k) and note the increased compute cost. `min_df` / `max_df` filters and embedding model for neural approaches also belong here.
3. Evaluation. Topic coherence (c_v) via `gensim.models.CoherenceModel`, topic diversity, and a 20-sample human read.
4. Failure mode to probe. For LDA, "junk topics" absorbing stopwords and frequent terms. For BERTopic, the -1 outlier cluster swallowing ambiguous documents.

Refuse BERTopic on documents longer than the embedding model's context window without a chunking strategy. Refuse LDA on very short text (tweets, reviews under 10 tokens) as coherence collapses. Flag any n_topics choice below 5 as likely wrong; flag >200 on corpora under 40k docs as likely over-splitting.
```

## 练习

1. **Easy.** Fit LDA 使用 5 主题 on the 20 Newsgroups dataset. Print top 10 词 per 主题. 标签 each 主题 by hand. Did the 算法 find the real categories?
2. 说明：**Medium.** Fit BERTopic on the same 20 Newsgroups subset. Compare the number of 主题 found, top 词, 与 qualitative coherence against LDA. 这 surfaces the real categories more cleanly?
3. **Hard.** 计算 c_v coherence 面向 both LDA 与 BERTopic on your 语料库. Run each 使用 5, 10, 20, 50 主题. Plot coherence vs 主题 count. Report 这 方法 is more stable across 主题 counts.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|主题|一个 thing the 语料库 is about|一个 概率 分布 over 词 (LDA) 或 a 聚类 of similar 文档 (BERTopic).|
|Mixed membership|Doc is multiple 主题|LDA assigns each 文档 a 分布 over all 主题.|
|UMAP|Dimensionality reduction|说明：Manifold learning 这 preserves local structure; used in BERTopic.|
|HDBSCAN|Density clustering|说明：Finds variable-size clusters; produces "噪声" 标签 (-1) 面向 outliers.|
|c_v coherence|主题 质量 指标|说明：Average pointwise mutual information of top 主题 词 within sliding windows.|

## 延伸阅读

- 说明：[说明：Blei, Ng, Jordan (2003). Latent Dirichlet Allocation](https://www.jmlr.org/papers/volume3/blei03a/blei03a.pdf)，the LDA paper.
- [说明：Grootendorst (2022). BERTopic: Neural 主题建模 使用 a class-based TF-IDF procedure](https://arxiv.org/abs/2203.05794)，the BERTopic paper.
- 说明：[说明：Röder, Both, Hinneburg (2015). Exploring the Space of 主题 Coherence Measures](https://svn.aksw.org/papers/2015/WSDM_Topic_Evaluation/public.pdf)，the paper 这 introduced c_v 与 friends.
- 说明：[BERTopic documentation](https://maartengr.github.io/BERTopic/)，the 生产 reference. Excellent examples.
