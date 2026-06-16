# 词嵌入，从零实现 Word2Vec

> 说明：一个 词 is the company it keeps. Train a shallow net on 这 idea 与 geometry falls out.

**类型:** 构建
**语言:** Python
**先修要求:** 说明：Phase 5 · 02 (BoW + TF-IDF), Phase 3 · 03 (Backpropagation 从 Scratch)
**时间:** 约75分钟

## 问题

说明：TF-IDF knows `dog` 与 `puppy` are different 词. It does not know they mean nearly the same thing. A classifier trained on `dog` cannot generalize to a review about `puppy`. You can paper over this by listing synonyms, but 这 fails on rare terms, 领域 jargon, 与 every language you did not anticipate.

You want a representation 其中 `dog` 与 `puppy` land close together in space. 其中 `king - man + woman` lands near `queen`. 其中 a 模型 trained on `dog` transfers some signal to `puppy` 面向 free.

说明：Word2Vec gave us 这 space. Two layer neural network, trillion-词元 训练 runs, published in 2013. The architecture is almost embarrassingly simple. The results reshaped NLP 面向 a decade.

## 概念

说明：**Distributional hypothesis** (Firth, 1957): "You shall know a 词 by the company it keeps." If two 词 appear in similar contexts, they probably mean similar things.

说明：Word2Vec comes in two flavors, both exploiting 这 idea.

- **Skip-gram.** 给定一个 center 词, predict the surrounding 词. `cat -> (the, sat, on)` 使用 window size 2.
- 说明：**CBOW (continuous bag of 词).** Given surrounding 词, predict the center. `(the, sat, on) -> cat`.

说明：Skip-gram is slower to train but handles rare 词 better. It became the 默认.

这个 network has one hidden layer 使用 no nonlinearity. Input is a one-hot 向量 over the vocabulary. 输出 is a softmax over the vocabulary. After 训练, you throw away the 输出 layer. The hidden layer weights are the 嵌入.

```
one-hot(center) ── W ──▶ hidden (d-dim) ── W' ──▶ softmax(vocab)
                          ^
                          this is the embedding
```

这个 trick: softmax over 100k 词 is prohibitively expensive. Word2Vec uses **negative sampling** to turn it 到 a binary 分类 任务. Predict "did this context 词 appear near this center 词, yes 或 no". Sample a handful of negative (non-co-occurring) 词 per 训练 pair instead of computing softmax over the whole vocabulary.

```figure
word-vector-arithmetic
```

## 动手构建

### Step 1: 训练 pairs 从 a 语料库

```python
def skipgram_pairs(docs, window=2):
    pairs = []
    for doc in docs:
        for i, center in enumerate(doc):
            for j in range(max(0, i - window), min(len(doc), i + window + 1)):
                if i == j:
                    continue
                pairs.append((center, doc[j]))
    return pairs
```

```python
>>> skipgram_pairs([["the", "cat", "sat", "on", "mat"]], window=2)
[('the', 'cat'), ('the', 'sat'),
 ('cat', 'the'), ('cat', 'sat'), ('cat', 'on'),
 ('sat', 'the'), ('sat', 'cat'), ('sat', 'on'), ('sat', 'mat'),
 ...]
```

说明：每个 (center, context) pair in a window is a positive 训练 example.

### Step 2: 嵌入 tables

说明：Two matrices. `W` is the center-词 嵌入 table (the one you keep). `W'` is the context-词 table (often discarded, sometimes averaged 使用 `W`).

```python
import numpy as np


def init_embeddings(vocab_size, dim, seed=0):
    rng = np.random.default_rng(seed)
    W = rng.normal(0, 0.1, size=(vocab_size, dim))
    W_prime = rng.normal(0, 0.1, size=(vocab_size, dim))
    return W, W_prime
```

说明：小 random init. Vocab size 10k 与 dim 100 is realistic; 面向 teaching, 50 vocab x 16 dim is enough to see the geometry.

### Step 3: negative sampling objective

对于 each positive pair `(center, context)`, sample `k` random 词 从 the vocabulary as negatives. Train the 模型 so the dot product `W[center] · W'[context]` is high 面向 positives 与 low 面向 negatives.

```python
def sigmoid(x):
    return 1.0 / (1.0 + np.exp(-np.clip(x, -20, 20)))


def train_pair(W, W_prime, center_idx, context_idx, negative_indices, lr):
    v_c = W[center_idx]
    u_pos = W_prime[context_idx]
    u_negs = W_prime[negative_indices]

    pos_score = sigmoid(v_c @ u_pos)
    neg_scores = sigmoid(u_negs @ v_c)

    grad_center = (pos_score - 1) * u_pos
    for i, u in enumerate(u_negs):
        grad_center += neg_scores[i] * u

    W[context_idx] = W[context_idx]
    W_prime[context_idx] -= lr * (pos_score - 1) * v_c
    for i, neg_idx in enumerate(negative_indices):
        W_prime[neg_idx] -= lr * neg_scores[i] * v_c
    W[center_idx] -= lr * grad_center
```

说明：这个 magic formula: logistic loss on positive pair (want sigmoid near 1) plus logistic loss on negative pairs (want sigmoid near 0). Gradients flow to both tables. Full derivation is in the original paper; walk through it once 使用 pencil 与 paper if you want it to stick.

### Step 4: train on a toy 语料库

```python
def train(docs, dim=16, window=2, k_neg=5, epochs=100, lr=0.05, seed=0):
    vocab = build_vocab(docs)
    vocab_size = len(vocab)
    rng = np.random.default_rng(seed)
    W, W_prime = init_embeddings(vocab_size, dim, seed=seed)
    pairs = skipgram_pairs(docs, window=window)

    for epoch in range(epochs):
        rng.shuffle(pairs)
        for center, context in pairs:
            c_idx = vocab[center]
            ctx_idx = vocab[context]
            negs = rng.integers(0, vocab_size, size=k_neg)
            negs = [n for n in negs if n != ctx_idx and n != c_idx]
            train_pair(W, W_prime, c_idx, ctx_idx, negs, lr)
    return vocab, W
```

After enough epochs on a 大 语料库, 词 这 share contexts have similar center 嵌入. On a toy 语料库, you see the effect faintly. On billions of 词元, you see it dramatically.

### Step 5: the analogy trick

```python
def nearest(vocab, W, target_vec, topk=5, exclude=None):
    exclude = exclude or set()
    inv_vocab = {i: w for w, i in vocab.items()}
    norms = np.linalg.norm(W, axis=1, keepdims=True) + 1e-9
    W_norm = W / norms
    target = target_vec / (np.linalg.norm(target_vec) + 1e-9)
    sims = W_norm @ target
    order = np.argsort(-sims)
    out = []
    for i in order:
        if i in exclude:
            continue
        out.append((inv_vocab[i], float(sims[i])))
        if len(out) == topk:
            break
    return out


def analogy(vocab, W, a, b, c, topk=5):
    v = W[vocab[b]] - W[vocab[a]] + W[vocab[c]]
    return nearest(vocab, W, v, topk=topk, exclude={vocab[a], vocab[b], vocab[c]})
```

On pre-trained 300d Google News 向量:

```python
>>> analogy(vocab, W, "man", "king", "woman")
[('queen', 0.71), ('monarch', 0.62), ('princess', 0.59), ...]
```

`king - man + woman = queen`. Not 因为 the 模型 knows what royalty is. 因为 the 向量 `(king - man)` captures something like "royal", 与 adding it to `woman` lands near the royal-female region.

## 投入使用

说明：Writing Word2Vec 从 scratch is teaching. Production NLP uses `gensim`.

```python
from gensim.models import Word2Vec

sentences = [
    ["the", "cat", "sat", "on", "the", "mat"],
    ["the", "dog", "ran", "across", "the", "room"],
]

model = Word2Vec(
    sentences,
    vector_size=100,
    window=5,
    min_count=1,
    sg=1,
    negative=5,
    workers=4,
    epochs=30,
)

print(model.wv["cat"])
print(model.wv.most_similar("cat", topn=3))
```

说明：对于 real work, you almost never train Word2Vec yourself. You download pre-trained 向量.

- 说明：**GloVe**，Stanford's co-occurrence-矩阵 factorization approach. 50d, 100d, 200d, 300d checkpoints. Good general coverage. Lesson 04 covers GloVe specifically.
- 说明：**fastText**，Facebook's Word2Vec extension 这 embeds character n-grams. Handles out-of-vocabulary 词 by composing subwords. Lesson 04.
- 说明：**Pretrained Word2Vec on Google News**，300d, 3M 词 vocabulary, published 2013. Still downloaded daily.

### 当 Word2Vec still wins in 2026

- Lightweight 领域-specific 检索. Train on medical abstracts in an hour on a laptop, get specialized 向量 no general 模型 captures.
- 说明：Analogy-style feature engineering. `gender_vector = mean(man - woman pairs)`. Subtract it 从 other 词 to get a gender-neutral axis. Still used in fairness research.
- 说明：Interpretability. 100d is 小 enough to plot via PCA 或 t-SNE 与 actually see clusters form.
- 说明：Anywhere 推理 has to run on-device 使用 no GPU. Word2Vec lookup is a single row fetch.

### 其中 Word2Vec fails

这个 polysemy wall. `bank` has one 向量. `river bank` 与 `financial bank` share it. `table` (spreadsheet vs. furniture) shares it. A classifier downstream cannot distinguish the senses 从 the 向量.

Contextual 嵌入 (ELMo, BERT, every transformer since) solved this by producing a different 向量 面向 each occurrence of the 词 based on surrounding context. 这 is the jump 从 Word2Vec to BERT: 从 static to contextual. Phase 7 covers the transformer half.

这个 out-of-vocabulary problem is the other failure. Word2Vec has never seen `Zoomer-approved` if it was not in 训练 数据. No fallback. fastText fixes this 使用 subword composition (lesson 04).

## 交付成果

保存为 `outputs/skill-embedding-probe.md`:

```markdown
---
name: embedding-probe
description: Inspect a word2vec model. Run analogies, find neighbors, diagnose quality.
version: 1.0.0
phase: 5
lesson: 03
tags: [nlp, embeddings, debugging]
---

You probe trained word embeddings to verify they are working. Given a `gensim.models.KeyedVectors` object and a vocabulary, you run:

1. Three canonical analogy tests. `king : man :: queen : woman`. `paris : france :: tokyo : japan`. `walking : walked :: swimming : ?`. Report the top-1 result and its cosine.
2. Five nearest-neighbor tests on domain-specific words the user supplies. Print top-5 neighbors with cosines.
3. One symmetry check. `similarity(a, b) == similarity(b, a)` to within float precision.
4. One degenerate check. If any embedding has a norm below 0.01 or above 100, the model has a training bug. Flag it.

Refuse to declare a model good on analogy accuracy alone. Analogy benchmarks are gameable and do not transfer to downstream tasks. Recommend intrinsic + downstream evaluation together.
```

## 练习

1. 说明：**Easy.** Run the 训练 loop on a tiny 语料库 (20 sentences about cats 与 dogs). After 200 epochs, verify `nearest(vocab, W, W[vocab["cat"]])` returns `dog` in its top 3. If not, increase epochs 或 vocabulary.
2. **Medium.** Add subsampling of frequent 词. 词 使用 frequency above `10^-5` are dropped 从 训练 pairs 使用 概率 proportional to their frequency. Measure the effect on rare-词 相似度.
3. **Hard.** Train a 模型 on the 20 Newsgroups 语料库. 计算 two bias axes: `he - she` 与 `doctor - nurse`. Project occupation 词 onto both axes. Report 这 occupations have the largest bias gap. This is the kind of probe fairness researchers use.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|词 嵌入|词 as a 向量|说明：一个 dense, low-dim (typically 100-300) representation learned 从 context.|
|Skip-gram|Word2Vec trick|说明：Predict context 词 从 center 词. Slower than CBOW, better 面向 rare 词.|
|Negative sampling|训练 shortcut|说明：Replace softmax over full vocab 使用 binary 分类 against `k` random 词.|
|Static 嵌入|One 向量 per 词|说明：Same 向量 regardless of context. Fails on polysemy.|
|Contextual 嵌入|Context-sensitive 向量|说明：Different 向量 面向 each occurrence based on surrounding 词. What transformers produce.|
|OOV|Out of vocabulary|词 not seen in 训练. Word2Vec cannot produce a 向量 面向 these.|

## 延伸阅读

- 说明：[说明：Mikolov et al. (2013). Distributed Representations of 词 与 Phrases 与 their Compositionality](https://arxiv.org/abs/1310.4546)，the negative-sampling paper. 短 与 readable.
- 说明：[说明：Rong, X. (2014). word2vec Parameter Learning Explained](https://arxiv.org/abs/1411.2738)，the clearest derivation of the gradients, if the original paper's math feels dense.
- 说明：[gensim Word2Vec tutorial](https://radimrehurek.com/gensim/models/word2vec.html)，生产 训练 settings 这 actually work.
