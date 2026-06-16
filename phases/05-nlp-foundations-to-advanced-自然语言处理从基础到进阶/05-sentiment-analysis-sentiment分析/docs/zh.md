# 情感分析

> 这个 canonical NLP 任务. Most of what you need to know about classical 文本 分类 shows up here.

**类型:** 构建
**语言:** Python
**先修要求:** Phase 5 · 02 (BoW + TF-IDF), Phase 2 · 14 (Naive Bayes)
**时间:** 约75分钟

## 问题

"The food was not great." Positive 或 negative?

Sentiment sounds simple. A reviewer said they liked 或 did not like something. 标签 the 句子. The 原因 it became the canonical NLP 任务 is 这 every easy-looking case hides a hard one. Negation flips meaning. Sarcasm inverts it. "Not bad at all" is positive despite two negative-coded 词. Emojis carry more signal than surrounding 文本. 领域 vocabulary matters (`tight` in music review versus `tight` in fashion review).

Sentiment is a working lab 面向 classical NLP. If you understand why every naive 基线 has a specific 失败模式, you understand why every richer 模型 was invented. This lesson builds a Naive Bayes 基线 从 scratch, adds logistic regression, 与 names the traps 这 make 生产 sentiment a compliance-grade problem.

## 概念

Classical sentiment is a two-step recipe.

1. 说明：**Represent.** Turn the 文本 到 a feature 向量. BoW, TF-IDF, 或 n-grams.
2. 说明：**Classify.** Fit a linear 模型 (Naive Bayes, logistic regression, SVM) on labeled examples.

Naive Bayes is the dumbest 模型 这 works. Assume every feature is independent given the 标签. Estimate `P(word | positive)` 与 `P(word | negative)` 从 counts. At 推理, multiply the probabilities. The "naive" independence assumption is laughably 错误 与 yet the results are shockingly strong. The 原因: 使用 sparse 文本 features 与 moderate 数据, the classifier cares about 这 side each 词 leans toward more than how much.

说明：Logistic regression fixes the independence assumption. It learns a weight per feature, including negative weights. `not good` as a bigram feature gets a negative weight. Naive Bayes cannot do 这 面向 bigrams it has never labeled.

```figure
sentiment-logits
```

## 动手构建

### Step 1: a real mini-dataset

```python
POSITIVE = [
    "absolutely loved this movie",
    "beautiful cinematography and a great story",
    "one of the best films of the year",
    "brilliant acting from the lead",
    "heartwarming and funny",
]

NEGATIVE = [
    "boring and far too long",
    "not worth your time",
    "the plot made no sense",
    "terrible acting, awful script",
    "i want my two hours back",
]
```

说明：小 on purpose. Real work uses tens of thousands of examples (IMDb, SST-2, Yelp polarity). The math is identical.

### Step 2: multinomial Naive Bayes 从 scratch

```python
import math
from collections import Counter


def train_nb(docs_by_class, vocab, alpha=1.0):
    class_priors = {}
    class_word_probs = {}
    total_docs = sum(len(d) for d in docs_by_class.values())

    for cls, docs in docs_by_class.items():
        class_priors[cls] = len(docs) / total_docs
        counts = Counter()
        for doc in docs:
            for token in doc:
                counts[token] += 1
        total = sum(counts.values()) + alpha * len(vocab)
        class_word_probs[cls] = {
            w: (counts[w] + alpha) / total for w in vocab
        }
    return class_priors, class_word_probs


def predict_nb(doc, class_priors, class_word_probs):
    scores = {}
    for cls in class_priors:
        s = math.log(class_priors[cls])
        for token in doc:
            if token in class_word_probs[cls]:
                s += math.log(class_word_probs[cls][token])
        scores[cls] = s
    return max(scores, key=scores.get)
```

Additive smoothing (alpha=1.0) is Laplace smoothing. 不使用 it, a 词 unseen in a class has 概率 zero 与 the log explodes. `alpha=0.01` is 常见 in practice. `alpha=1.0` is the teaching 默认.

### Step 3: logistic regression 从 scratch

```python
import numpy as np


def sigmoid(x):
    return 1.0 / (1.0 + np.exp(-np.clip(x, -20, 20)))


def train_lr(X, y, epochs=500, lr=0.05, l2=0.01):
    n_features = X.shape[1]
    w = np.zeros(n_features)
    b = 0.0
    for _ in range(epochs):
        logits = X @ w + b
        preds = sigmoid(logits)
        err = preds - y
        grad_w = X.T @ err / len(y) + l2 * w
        grad_b = err.mean()
        w -= lr * grad_w
        b -= lr * grad_b
    return w, b


def predict_lr(X, w, b):
    return (sigmoid(X @ w + b) >= 0.5).astype(int)
```

L2 regularization matters here. 文本 features are sparse; 不使用 L2 the 模型 memorizes 训练 examples. Start at `0.01` 与 tune.

### Step 4: handling negation (the 失败模式)

Consider "not good" 与 "not bad". A BoW classifier sees `{not, good}` 与 `{not, bad}` 与 learns 从 whichever showed up more in 训练. A bigram classifier sees `not_good` 与 `not_bad` 与 learns them as distinct features. 这 is usually enough.

一个 cruder fix 这 works when you do not have bigrams: **negation scoping**. Prefix 词元 following a negation 词 使用 `NOT_` up to the next punctuation.

```python
NEGATION_WORDS = {"not", "no", "never", "nor", "none", "nothing", "neither"}
NEGATION_TERMINATORS = {".", "!", "?", ",", ";"}


def apply_negation(tokens):
    out = []
    negate = False
    for token in tokens:
        if token in NEGATION_TERMINATORS:
            negate = False
            out.append(token)
            continue
        if token in NEGATION_WORDS:
            negate = True
            out.append(token)
            continue
        out.append(f"NOT_{token}" if negate else token)
    return out
```

```python
>>> apply_negation(["not", "good", "at", "all", ".", "but", "funny"])
['not', 'NOT_good', 'NOT_at', 'NOT_all', '.', 'but', 'funny']
```

说明：Now `good` 与 `NOT_good` are different features. The classifier can weight them opposite. Three lines of preprocessing, measurable 准确率 jump on sentiment benchmarks.

### Step 5: 评估 metrics 这 matter

准确率 alone is misleading if classes are imbalanced. Real sentiment corpora are usually 70-80% positive 或 70-80% negative; a constant-majority classifier gets 80% 准确率 与 is worthless. Report every one of the following:

- **Per-class 精确率 与 召回率.** One pair per class. Macro-average them to get a single number 这 respects class balance.
- **Macro-F1 (primary 指标 面向 imbalanced 数据).** Mean of per-class F1 scores, equally weighted. Use this instead of 准确率 when classes are imbalanced.
- 说明：**Weighted-F1 (alternative).** Same as macro but weighted by class frequency. Report alongside macro-F1 when the imbalance itself has business meaning.
- **Confusion 矩阵.** 原始 counts. Always inspect before trusting any scalar 指标; it reveals 这 pair of classes the 模型 confuses.
- 说明：**Per-class error samples.** Pull 5 错误 predictions per class. Read them. Nothing replaces reading the actual errors.

对于 severely imbalanced 数据 (> 95-5 ratio), report **AUROC** 与 **AUPRC** instead of 准确率. AUPRC is more sensitive to the minority class, 这 is what you usually care about (spam, fraud, rare sentiment).

说明：**常见 bug to avoid.** Reporting micro-F1 instead of macro-F1 on imbalanced 数据 gives a number 这 looks high 因为 it is dominated by the majority class. Macro-F1 forces you to see the minority-class performance.

```python
def evaluate(y_true, y_pred):
    tp = sum(1 for t, p in zip(y_true, y_pred) if t == 1 and p == 1)
    fp = sum(1 for t, p in zip(y_true, y_pred) if t == 0 and p == 1)
    fn = sum(1 for t, p in zip(y_true, y_pred) if t == 1 and p == 0)
    tn = sum(1 for t, p in zip(y_true, y_pred) if t == 0 and p == 0)
    precision = tp / (tp + fp) if tp + fp else 0
    recall = tp / (tp + fn) if tp + fn else 0
    f1 = 2 * precision * recall / (precision + recall) if precision + recall else 0
    return {"tp": tp, "fp": fp, "tn": tn, "fn": fn, "precision": precision, "recall": recall, "f1": f1}
```

## 投入使用

说明：scikit-learn does it in six lines, correctly.

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.pipeline import Pipeline

pipe = Pipeline([
    ("tfidf", TfidfVectorizer(ngram_range=(1, 2), min_df=2, sublinear_tf=True, stop_words=None)),
    ("clf", LogisticRegression(C=1.0, max_iter=1000)),
])
pipe.fit(X_train, y_train)
print(pipe.score(X_test, y_test))
```

Three things to notice. `stop_words=None` keeps negations. `ngram_range=(1, 2)` adds bigrams so `not_good` becomes a feature. `sublinear_tf=True` dampens repeated 词. These three flags are the difference between a 75%-准确 基线 与 an 85%-准确 基线 on SST-2.

### 当 to reach 面向 a transformer

- 说明：Sarcasm detection. Classical models fail here. Period.
- 长 reviews 其中 sentiment shifts mid-文档.
- 说明：Aspect-based sentiment. "Camera was great but battery was terrible." You need to attribute sentiment to aspects. Transformers 或 structured 输出 models only.
- 说明：Non-English, low-resource languages. Multilingual BERT gives you a zero-shot 基线 面向 free.

说明：如果 you need any of the above, skip ahead to phase 7 (transformers deep dive). Otherwise, Naive Bayes 或 logistic regression on TF-IDF plus bigrams plus negation handling is your 2026 生产 基线.

### 这个 reproducibility trap (again)

Retraining sentiment models is routine. Re-evaluating them is not. 准确率 numbers reported in papers use specific splits, specific preprocessing, specific tokenizers. If you compare your new 模型 to a 基线 不使用 using the identical 流水线, you will get misleading deltas. Always regenerate the 基线 on your 流水线, not the paper's number.

## 交付成果

保存为 `outputs/prompt-sentiment-baseline.md`:

```markdown
---
name: sentiment-baseline
description: Design a sentiment analysis baseline for a new dataset.
phase: 5
lesson: 05
---

Given a dataset description (domain, language, size, label granularity, latency budget), you output:

1. Feature extraction recipe. Specify tokenizer, n-gram range, stopword policy (usually keep), negation handling (scoped prefix or bigrams).
2. Classifier. Naive Bayes for baseline, logistic regression for production, transformer only if the domain needs sarcasm / aspects / cross-lingual.
3. Evaluation plan. Report precision, recall, F1, confusion matrix, and per-class error samples (not just scalars).
4. One failure mode to monitor post-deployment. Domain drift and sarcasm are the top two.

Refuse to recommend dropping stopwords for sentiment tasks. Refuse to report accuracy as the sole metric when classes are imbalanced (e.g., 90% positive). Flag subword-rich languages as needing FastText or transformer embeddings over word-level TF-IDF.
```

## 练习

1. 说明：**Easy.** Add `apply_negation` as a preprocessing step in the scikit-learn 流水线 与 measure the F1 delta on a 小 sentiment dataset.
2. 说明：**Medium.** Implement class-weighted logistic regression (pass `class_weight="balanced"` to scikit-learn, 或 derive the gradient yourself). Measure the effect on a synthetic 90-10 class imbalance.
3. **Hard.** Build a sarcasm detector by 训练 a second classifier on the residuals of the sentiment 模型. 文档 your experimental setup. Warn the reader when your 准确率 is below chance (chance-level on 2-class sarcasm is ~50%, 与 most first attempts land there).

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|Polarity|Positive 或 negative|说明：Binary 标签; sometimes extended to neutral 或 fine-grained (5-star).|
|Aspect-based sentiment|Per-aspect polarity|说明：Attribute sentiment to specific entities 或 attributes mentioned in 文本.|
|Negation scoping|Reversing nearby 词元|Prefix 词元 after "not" 使用 `NOT_` until punctuation.|
|Laplace smoothing|Adding 1 to counts|Prevents zero-概率 features in Naive Bayes.|
|L2 regularization|Shrinking weights|Adds `lambda * sum(w^2)` to loss. Essential 面向 sparse 文本 features.|

## 延伸阅读

- 说明：[说明：Pang 与 Lee (2008). Opinion Mining 与 Sentiment Analysis](https://www.cs.cornell.edu/home/llee/opinion-mining-sentiment-analysis-survey.html)，the foundational survey. 长, but the first four sections cover everything classical.
- [说明：Wang 与 Manning (2012). Baselines 与 Bigrams: Simple, Good Sentiment 与 主题 Classification](https://aclanthology.org/P12-2018/)，the paper 这 showed bigrams + Naive Bayes is hard to beat on 短 文本.
- 说明：[scikit-learn 文本 feature extraction docs](https://scikit-learn.org/stable/modules/feature_extraction.html#text-feature-extraction)，reference 面向 `CountVectorizer`, `TfidfVectorizer`, 与 every knob you'll tune.
