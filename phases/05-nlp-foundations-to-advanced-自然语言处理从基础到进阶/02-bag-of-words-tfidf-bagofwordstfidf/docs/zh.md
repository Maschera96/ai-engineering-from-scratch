# 词袋、TF-IDF 与文本表示

> 说明：Count first, think later. TF-IDF still beats 嵌入 on well-defined tasks in 2026.

**类型:** 构建
**语言:** Python
**先修要求:** 说明：Phase 5 · 01 (文本 Processing), Phase 2 · 02 (Linear Regression 从 Scratch)
**时间:** 约75分钟

## 问题

这个 模型 needs numbers. You have strings.

每个 NLP 流水线 has to 答案 the same 问题. How do we turn a variable-length stream of 词元 到 a 固定-size 向量 这 a classifier can consume. The first 答案 the field landed on was the dumbest one 这 works. Count the 词. Make a 向量.

这 向量 has carried more 生产 NLP than any 嵌入 模型. Spam filters, 主题 classifiers, log anomaly detection, search ranking (before BM25), the first wave of sentiment analysis, the first decade of academic NLP benchmarks. 2026 practitioners still reach 面向 it first on narrow 分类 tasks. It is 快, interpretable, 与 often indistinguishable 从 a 400M-parameter 嵌入 模型 on tasks 其中 词 presence is what matters.

本课 builds bag of 词, then TF-IDF, 从 scratch. Then shows scikit-learn doing the same in three lines. Then names the 失败模式 这 makes you reach 面向 嵌入.

## 概念

说明：**Bag of 词 (BoW)** throws away order. 面向 each 文档, count how many times each vocabulary 词 appears. Vector length is the vocabulary size. Position `i` is the count of 词 `i`.

**TF-IDF** reweights BoW. A 词 这 appears in every 文档 is uninformative, so scale it down. A 词 rare across the 语料库 but frequent in a single 文档 is signal, so scale it up.

```
TF-IDF(w, d) = TF(w, d) * IDF(w)
             = count(w in d) / |d| * log(N / df(w))
```

其中 `TF` is term frequency in the 文档, `df` is 文档 frequency (how many docs contain the 词), `N` is total 文档. The `log` keeps the weight bounded 面向 ubiquitous 词.

关键 property: both produce sparse 向量 使用 interpretable axes. You can look at a trained classifier's weights 与 read 这 词 push a 文档 toward each class. You cannot do this 使用 a 768-dimensional BERT 嵌入.

```figure
bow-tfidf
```

## 动手构建

### Step 1: build the vocabulary

```python
def build_vocab(docs):
    vocab = {}
    for doc in docs:
        for token in doc:
            if token not in vocab:
                vocab[token] = len(vocab)
    return vocab
```

Input: list of tokenized 文档 (any 词-level 分词器 will do; the `code/main.py` in this lesson uses a simplified lowercase variant). 输出: `{word: index}` dict. Stable insertion order means 词 index 0 is the first 词 seen in the first 文档. Convention varies; scikit-learn sorts alphabetically.

### Step 2: bag of 词

```python
def bag_of_words(docs, vocab):
    matrix = [[0] * len(vocab) for _ in docs]
    for i, doc in enumerate(docs):
        for token in doc:
            if token in vocab:
                matrix[i][vocab[token]] += 1
    return matrix
```

```python
>>> docs = [["cat", "sat", "on", "mat"], ["cat", "cat", "ran"]]
>>> vocab = build_vocab(docs)
>>> bag_of_words(docs, vocab)
[[1, 1, 1, 1, 0], [2, 0, 0, 0, 1]]
```

Rows are 文档. Columns are vocabulary indices. Entry `[i][j]` is "how many times 词 `j` appears in 文档 `i`." Doc 1 has `cat` twice 因为 it did. Doc 0 has `ran` zero times 因为 it did not.

### Step 3: term frequency 与 文档 frequency

```python
import math


def term_frequency(doc_bow, doc_length):
    return [c / doc_length if doc_length else 0 for c in doc_bow]


def document_frequency(bow_matrix):
    df = [0] * len(bow_matrix[0])
    for row in bow_matrix:
        for j, count in enumerate(row):
            if count > 0:
                df[j] += 1
    return df


def inverse_document_frequency(df, n_docs):
    return [math.log((n_docs + 1) / (d + 1)) + 1 for d in df]
```

说明：Two smoothing tricks worth naming. The `(n+1)/(d+1)` avoids `log(x/0)`. The trailing `+1` ensures a 词 in every 文档 still has IDF 1 (not 0), matching scikit-learn's 默认. Other implementations use 原始 `log(N/df)`. Both work; the smoothed version is friendlier.

### Step 4: TF-IDF

```python
def tfidf(bow_matrix):
    n_docs = len(bow_matrix)
    df = document_frequency(bow_matrix)
    idf = inverse_document_frequency(df, n_docs)
    out = []
    for row in bow_matrix:
        length = sum(row)
        tf = term_frequency(row, length)
        out.append([tf_j * idf_j for tf_j, idf_j in zip(tf, idf)])
    return out
```

```python
>>> docs = [
...     ["the", "cat", "sat"],
...     ["the", "dog", "sat"],
...     ["the", "cat", "ran"],
... ]
>>> vocab = build_vocab(docs)
>>> bow = bag_of_words(docs, vocab)
>>> tfidf(bow)
```

Three 文档, five vocab 词 (`the`, `cat`, `sat`, `dog`, `ran`). `the` appears in all three, so its IDF is low. `dog` appears in one, so its IDF is high. The 向量 are sparse (most entries are 小) 与 the discriminative 词 pop.

### Step 5: L2-normalize rows

```python
def l2_normalize(matrix):
    out = []
    for row in matrix:
        norm = math.sqrt(sum(x * x for x in row))
        out.append([x / norm if norm else 0 for x in row])
    return out
```

不使用 normalization, a longer 文档 gets a larger 向量 与 dominates 相似度 scores. L2 normalization puts every 文档 on the unit hypersphere. Cosine 相似度 between rows is now just a dot product.

## 投入使用

scikit-learn ships the 生产 version.

```python
from sklearn.feature_extraction.text import CountVectorizer, TfidfVectorizer

docs = ["the cat sat on the mat", "the dog sat on the mat", "the cat ran"]

bow_vectorizer = CountVectorizer()
bow = bow_vectorizer.fit_transform(docs)
print(bow_vectorizer.get_feature_names_out())
print(bow.toarray())

tfidf_vectorizer = TfidfVectorizer()
tfidf = tfidf_vectorizer.fit_transform(docs)
print(tfidf.toarray().round(3))
```

`CountVectorizer` does 分词, vocabulary, 与 BoW in one call. `TfidfVectorizer` adds IDF weighting 与 L2 normalization. Both return sparse matrices. 面向 100k 文档, the dense version does not fit in memory; stay sparse until the classifier demands dense.

Knobs 这 change everything:

|Arg|Effect|
|-----|--------|
|`ngram_range=(1, 2)`|Include bigrams. Usually boosts 分类.|
|`min_df=2`|说明：Drop 词 in fewer than 2 docs. Trims vocabulary on noisy 数据.|
|`max_df=0.95`|说明：Drop 词 in more than 95% of docs. Approximates stopword removal 不使用 a hardcoded list.|
|`stop_words="english"`|说明：scikit-learn's builtin stopword list. 任务-dependent，sentiment analysis should *not* drop negations.|
|`sublinear_tf=True`|说明：Use `1 + log(tf)` instead of 原始 `tf`. Helps when a term repeats many times in one doc.|

### 当 TF-IDF still wins (as of 2026)

- 说明：Spam detection, 主题 labeling, log anomaly flagging. 词 presence is what matters; 语义 nuance does not.
- 说明：Low-数据 regimes (hundreds of labeled examples). TF-IDF plus logistic regression has no pretraining 成本.
- Anywhere 延迟 matters. TF-IDF plus a linear 模型 answers in microseconds. 嵌入 a 文档 through a transformer takes 10-100ms.
- 说明：Systems 这 must explain their predictions. Inspect the classifier's coefficients. Top positive 词 are the 原因.

### 当 TF-IDF fails

这个 语义 blindness failure. Consider these two 文档:

- "The movie was not good at all."
- "The movie was excellent."

One is a negative review. One is positive. Their TF-IDF overlap is exactly `{the, movie, was}`. A bag-of-词 classifier has to memorize 这 the 词 `not` near `good` flips the 标签. It can learn this on enough 数据, but never as gracefully as a 模型 这 understands syntax.

这个 other failure: out-of-vocabulary 词 at 推理. A BoW 模型 trained on IMDb reviews has no idea what to do 使用 `Zoomer-approved` if 这 词元 never appeared in 训练. Subword 嵌入 (lesson 04) handle this. TF-IDF cannot.

### Hybrid: TF-IDF weighted 嵌入

这个 2026 pragmatic 默认 面向 medium-数据 分类: use TF-IDF weights as 注意力 over 词 嵌入.

```python
def tfidf_weighted_embedding(doc, tfidf_scores, embedding_table, dim):
    vec = [0.0] * dim
    total_weight = 0.0
    for token in doc:
        if token not in embedding_table or token not in tfidf_scores:
            continue
        weight = tfidf_scores[token]
        emb = embedding_table[token]
        for i in range(dim):
            vec[i] += weight * emb[i]
        total_weight += weight
    if total_weight == 0:
        return vec
    return [v / total_weight for v in vec]
```

You get 语义 capacity 从 嵌入, 与 rare-词 emphasis 从 TF-IDF. Classifier trains on the pooled 向量. This outperforms either on its own 面向 sentiment, 主题, 与 intent 分类 below about 50k labeled examples.

## 交付成果

保存为 `outputs/prompt-vectorization-picker.md`:

```markdown
---
name: vectorization-picker
description: Given a text-classification task, recommend BoW, TF-IDF, embeddings, or a hybrid.
phase: 5
lesson: 02
---

You recommend a text-vectorization strategy. Given a task description, output:

1. Representation (BoW, TF-IDF, transformer embeddings, or a hybrid). Explain why in one sentence.
2. Specific vectorizer configuration. Name the library. Quote the arguments (`ngram_range`, `min_df`, `max_df`, `sublinear_tf`, `stop_words`).
3. One failure mode to test before shipping.

Refuse to recommend embeddings when the user has under 500 labeled examples unless they show evidence of semantic failure in a TF-IDF baseline. Refuse to remove stopwords for sentiment analysis (negations carry signal). Flag class imbalance as needing more than a vectorizer change.

Example input: "Classifying 30k customer support tickets into 12 categories. Most tickets are 2-3 sentences. English only. Need explainability for audit logs."

Example output:

- Representation: TF-IDF. 30k examples is not small; explainability requirement rules out dense embeddings.
- Config: `TfidfVectorizer(ngram_range=(1, 2), min_df=3, max_df=0.95, sublinear_tf=True, stop_words=None)`. Keep stopwords because category keywords sometimes are stopwords ("not working" vs "working").
- Failure to test: verify `min_df=3` does not drop rare category keywords. Run `get_feature_names_out` filtered by class and eyeball.
```

## 练习

1. **Easy.** Implement `cosine_similarity(doc_vec_a, doc_vec_b)` on the L2-normalized TF-IDF 输出. Verify 这 identical 文档 score 1.0 与 disjoint-vocabulary 文档 score 0.0.
2. 说明：**Medium.** Add `n-gram` support to `bag_of_words`. Parameter `n` produces counts over `n`-grams. Test 这 `n=2` on `["the", "cat", "sat"]` produces bigram counts 面向 `["the cat", "cat sat"]`.
3. **Hard.** Build the TF-IDF-weighted-嵌入 hybrid above using GloVe 100d 向量 (download once, cache). Compare 分类 准确率 against plain TF-IDF 与 plain mean-pooled 嵌入 on the 20 Newsgroups dataset. Report 这 wins 其中.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|BoW|词 frequency 向量|说明：Counts of vocabulary 词 in one 文档. Throws away order.|
|TF|Term frequency|说明：Count of a 词 in a 文档, optionally normalized by 文档 length.|
|DF|文档 frequency|Count of 文档 containing the 词 at least once.|
|IDF|Inverse 文档 frequency|说明：`log(N / df)` smoothed. Downweights 词 这 appear everywhere.|
|Sparse 向量|Mostly zeros|说明：Vocabulary is typically 10k-100k 词; most are absent 从 any given 文档.|
|Cosine 相似度|Vector angle|说明：Dot product of L2-normalized 向量. 1 is identical, 0 is orthogonal.|

## 延伸阅读

- 说明：[scikit-learn，feature extraction 从 文本](https://scikit-learn.org/stable/modules/feature_extraction.html#text-feature-extraction)，the canonical API reference, plus notes on every knob.
- [说明：Salton, G., & Buckley, C. (1988). Term-weighting approaches in automatic 文本 检索](https://www.sciencedirect.com/science/article/pii/0306457388900210)，the paper 这 made TF-IDF the 默认 面向 a decade.
- 说明：[说明："Why TF-IDF Still Beats 嵌入"，Ashfaque Thonikkadavan (Medium)](https://medium.com/@cmtwskb/why-tf-idf-still-beats-embeddings-ad85c123e1b2)，2026 take on when the old 方法 wins 与 why.
