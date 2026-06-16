# GloVe、FastText 与子词嵌入

> 说明：Word2Vec trained one 嵌入 per 词. GloVe factorized the co-occurrence 矩阵. FastText embedded the pieces. BPE bridged to transformers.

**类型:** 构建
**语言:** Python
**先修要求:** Phase 5 · 03 (Word2Vec 从 Scratch)
**时间:** 约45分钟

## 问题

Word2Vec left two 开放 questions.

First, there was a parallel line of research 这 factorized the co-occurrence 矩阵 directly (LSA, HAL) rather than doing online skip-gram updates. Was Word2Vec's iterative approach fundamentally better, 或 was the difference an artifact of how the two methods handled counts? **GloVe** answered 这: 矩阵 factorization 使用 a thoughtfully chosen loss matches 或 beats Word2Vec, 与 costs less to train.

Second, neither 方法 had a story 面向 词 it had never seen. `Zoomer-approved`, `dogecoin`, any proper noun coined last week, every inflected form of a rare root. **FastText** 固定 this by 嵌入 character n-grams: a 词 is the sum of its parts, including morphemes, so even out-of-vocabulary 词 get a sensible 向量.

Third, once transformers arrived, the 问题 shifted again. 词-level vocabularies cap out around a million entries; real language is more 开放 than 这. **Byte-pair encoding (BPE)** 与 its relatives solved this by learning a vocabulary of frequent subword units 这 covers everything. Every 现代 分词器 面向 every 现代 LLM is a subword 分词器.

说明：本课 walks all three, then explains 这 to reach 面向 when.

## 概念

**GloVe (Global Vectors).** Build the 词-词 co-occurrence 矩阵 `X` 其中 `X[i][j]` is how often 词 `j` appears in the context of 词 `i`. Train 向量 such 这 `v_i · v_j + b_i + b_j ≈ log(X[i][j])`. Weight the loss so frequent pairs do not dominate. Done.

**FastText.** A 词 is the sum of its character n-grams plus the 词 itself. `where` becomes `<wh, whe, her, ere, re>, <where>`. The 词 向量 is the sum of those component 向量. Train as Word2Vec. Benefit: unseen 词 (`whereupon`) compose 从 known n-grams.

**BPE (Byte-Pair Encoding).** Start 使用 a vocabulary of individual bytes (或 characters). Count every adjacent pair in the 语料库. Merge the most frequent pair 到 a new 词元. Repeat 面向 `k` iterations. Result: a vocabulary of `k + 256` 词元 其中 frequent sequences (`ing`, `tion`, `the`) are single 词元 与 rare 词 are broken 到 familiar pieces. Every 句子 tokenizes 到 something.

## 动手构建

### GloVe: factorize the co-occurrence 矩阵

```python
import numpy as np
from collections import Counter


def build_cooccurrence(docs, window=5):
    pair_counts = Counter()
    vocab = {}
    for doc in docs:
        for token in doc:
            if token not in vocab:
                vocab[token] = len(vocab)
    for doc in docs:
        indexed = [vocab[t] for t in doc]
        for i, center in enumerate(indexed):
            for j in range(max(0, i - window), min(len(indexed), i + window + 1)):
                if i != j:
                    distance = abs(i - j)
                    pair_counts[(center, indexed[j])] += 1.0 / distance
    return vocab, pair_counts


def glove_train(vocab, pair_counts, dim=16, epochs=100, lr=0.05, x_max=100, alpha=0.75, seed=0):
    n = len(vocab)
    rng = np.random.default_rng(seed)
    W = rng.normal(0, 0.1, size=(n, dim))
    W_tilde = rng.normal(0, 0.1, size=(n, dim))
    b = np.zeros(n)
    b_tilde = np.zeros(n)

    for epoch in range(epochs):
        for (i, j), x_ij in pair_counts.items():
            weight = (x_ij / x_max) ** alpha if x_ij < x_max else 1.0
            diff = W[i] @ W_tilde[j] + b[i] + b_tilde[j] - np.log(x_ij)
            coef = weight * diff

            grad_W_i = coef * W_tilde[j]
            grad_W_tilde_j = coef * W[i]
            W[i] -= lr * grad_W_i
            W_tilde[j] -= lr * grad_W_tilde_j
            b[i] -= lr * coef
            b_tilde[j] -= lr * coef

    return W + W_tilde
```

说明：Two moving pieces worth naming. The weighting 函数 `f(x) = (x/x_max)^alpha` downweights very frequent pairs (like `(the, and)`) so they do not dominate the loss. The final 嵌入 is the sum of `W` (center) 与 `W_tilde` (context) tables. Summing both is a published trick 这 tends to outperform using just one.

### FastText: subword-aware 嵌入

```python
def char_ngrams(word, n_min=3, n_max=6):
    wrapped = f"<{word}>"
    grams = {wrapped}
    for n in range(n_min, n_max + 1):
        for i in range(len(wrapped) - n + 1):
            grams.add(wrapped[i:i + n])
    return grams
```

```python
>>> char_ngrams("where")
{'<where>', '<wh', 'whe', 'her', 'ere', 're>', '<whe', 'wher', 'here', 'ere>', '<wher', 'where', 'here>'}
```

每个 词 is represented by its set of n-grams (typically 3 to 6 characters). The 词 嵌入 is the sum of its n-gram 嵌入. 面向 skip-gram 训练, plug this in 其中 Word2Vec used a single 向量.

```python
def fasttext_vector(word, ngram_table):
    grams = char_ngrams(word)
    vecs = [ngram_table[g] for g in grams if g in ngram_table]
    if not vecs:
        return None
    return np.sum(vecs, axis=0)
```

对于 an unseen 词, you still get a 向量 as 长 as some of its n-grams are known. `whereupon` shares `<wh`, `her`, `ere`, 与 `<where` 使用 `where`, so the two land near each other.

### BPE: learned subword vocabulary

```python
def learn_bpe(corpus, k_merges):
    vocab = Counter()
    for word, freq in corpus.items():
        tokens = tuple(word) + ("</w>",)
        vocab[tokens] = freq

    merges = []
    for _ in range(k_merges):
        pair_freq = Counter()
        for tokens, freq in vocab.items():
            for a, b in zip(tokens, tokens[1:]):
                pair_freq[(a, b)] += freq
        if not pair_freq:
            break
        best = pair_freq.most_common(1)[0][0]
        merges.append(best)

        new_vocab = Counter()
        for tokens, freq in vocab.items():
            new_tokens = []
            i = 0
            while i < len(tokens):
                if i + 1 < len(tokens) and (tokens[i], tokens[i + 1]) == best:
                    new_tokens.append(tokens[i] + tokens[i + 1])
                    i += 2
                else:
                    new_tokens.append(tokens[i])
                    i += 1
            new_vocab[tuple(new_tokens)] = freq
        vocab = new_vocab
    return merges


def apply_bpe(word, merges):
    tokens = list(word) + ["</w>"]
    for a, b in merges:
        new_tokens = []
        i = 0
        while i < len(tokens):
            if i + 1 < len(tokens) and tokens[i] == a and tokens[i + 1] == b:
                new_tokens.append(a + b)
                i += 2
            else:
                new_tokens.append(tokens[i])
                i += 1
        tokens = new_tokens
    return tokens
```

```python
>>> corpus = Counter({"low": 5, "lower": 2, "newest": 6, "widest": 3})
>>> merges = learn_bpe(corpus, k_merges=10)
>>> apply_bpe("lowest", merges)
['low', 'est</w>']
```

说明：First iteration merges the most 常见 adjacent pair. After enough iterations, frequent substrings (`low`, `est`, `tion`) become single 词元 与 rare 词 break cleanly.

说明：这个 real GPT / BERT / T5 tokenizers learn 30k-100k merges. Result: any 文本 tokenizes 到 a bounded-length 序列 of known IDs, no OOV ever.

## 投入使用

说明：在 practice, you rarely train any of these yourself. You load pre-trained checkpoints.

```python
import fasttext.util
fasttext.util.download_model("en", if_exists="ignore")
ft = fasttext.load_model("cc.en.300.bin")
print(ft.get_word_vector("whereupon").shape)
print(ft.get_word_vector("zoomerapproved").shape)
```

对于 BPE-style subword 分词 in the transformer era:

```python
from transformers import AutoTokenizer

tok = AutoTokenizer.from_pretrained("gpt2")
print(tok.tokenize("unbelievably tokenized"))
```

```
['un', 'bel', 'iev', 'ably', 'Ġtoken', 'ized']
```

这个 `Ġ` prefix marks 词 boundaries (a GPT-2 convention). Every 现代 分词器 is a BPE variant, WordPiece (BERT), 或 SentencePiece (T5, LLaMA).

### 如何选择

|Situation|Pick|
|-----------|------|
|说明：Pretrained general-purpose 词 向量, no OOV tolerance needed|GloVe 300d|
|说明：Pretrained general-purpose 词 向量, must handle misspellings / neologisms / morphologically rich languages|FastText|
|Anything going 到 a transformer (训练 或 推理)|Whatever 分词器 the 模型 shipped 使用. Never swap.|
|训练 your own 语言模型 从 scratch|Train a BPE 或 SentencePiece 分词器 on your 语料库 first|
|Production 文本 分类 使用 a linear 模型|Still TF-IDF. Lesson 02.|

## 交付成果

保存为 `outputs/skill-embeddings-picker.md`:

```markdown
---
name: tokenizer-picker
description: Pick a tokenization approach for a new language model or text pipeline.
version: 1.0.0
phase: 5
lesson: 04
tags: [nlp, tokenization, embeddings]
---

Given a task and dataset description, you output:

1. Tokenization strategy (word-level, BPE, WordPiece, SentencePiece, byte-level). One-sentence reason.
2. Vocabulary size target (e.g., 32k for an English-only LM, 64k-100k for multilingual).
3. Library call with the exact training command. Name the library. Quote the arguments.
4. One reproducibility pitfall. Tokenizer-model mismatch is the single most common silent production bug; call out which pair must be used together.

Refuse to recommend training a custom tokenizer when the user is fine-tuning a pretrained LLM. Refuse to recommend word-level tokenization for any model targeting production inference. Flag non-English / multi-script corpora as needing SentencePiece with byte fallback.
```

## 练习

1. 说明：**Easy.** Run `char_ngrams("playing")` 与 `char_ngrams("played")`. 计算 the Jaccard overlap of the two n-gram sets. You should see substantial shared pieces (`pla`, `lay`, `play`), 这 is why FastText transfers well across morphological variants.
2. **Medium.** Extend `learn_bpe` to track vocabulary growth. Plot 词元-per-语料库-character as a 函数 of number of merges. You should see rapid compression at first, asymptoting near ~2-3 chars per 词元.
3. **Hard.** Train a 1k-merge BPE on Shakespeare's complete works. Compare 分词 of 常见 词 vs. rare proper nouns. Measure average 词元 per 词 before 与 after. Write up what surprised you.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|Co-occurrence 矩阵|词-词 frequency table|`X[i][j]` = how often 词 `j` appears in a window around 词 `i`.|
|Subword|Piece of a 词|说明：一个 character n-gram (FastText) 或 learned 词元 (BPE/WordPiece/SentencePiece).|
|BPE|Byte-pair encoding|说明：Iterative merging of most-frequent adjacent pairs until vocabulary hits target size.|
|OOV|Out of vocabulary|说明：词 the 模型 has never seen. Word2Vec/GloVe fail. FastText 与 BPE handle it.|
|Byte-level BPE|BPE on 原始 bytes|说明：GPT-2's scheme. Vocabulary starts 使用 256 bytes, so nothing is ever OOV.|

## 延伸阅读

- 说明：[说明：Pennington, Socher, Manning (2014). GloVe: Global Vectors 面向 词 Representation](https://nlp.stanford.edu/pubs/glove.pdf)，the GloVe paper, seven pages, still the best derivation of the loss.
- 说明：[说明：Bojanowski et al. (2017). Enriching 词 Vectors 使用 Subword Information](https://arxiv.org/abs/1607.04606)，FastText.
- [说明：Sennrich, Haddow, Birch (2016). Neural Machine Translation of Rare 词 使用 Subword Units](https://arxiv.org/abs/1508.07909)，the paper 这 introduced BPE to 现代 NLP.
- 说明：[Hugging Face 分词器 摘要](https://huggingface.co/docs/transformers/tokenizer_summary)，how BPE, WordPiece, 与 SentencePiece actually differ in practice.
