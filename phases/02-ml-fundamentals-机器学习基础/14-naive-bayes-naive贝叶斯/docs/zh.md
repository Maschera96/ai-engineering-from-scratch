# 朴素贝叶斯

> The "naive" assumption is wrong, and it works anyway. That's the beauty of it.

**Type:** 构建
**Language:** Python
**Prerequisites:** Phase 2, Lessons 01-07 (分类, Bayes' theorem)
**Time:** ~75 分钟

## 学习目标

- 实现 Multinomial 朴素贝叶斯 从零实现 with Laplace smoothing for text 分类
- 解释 why the naive independence assumption is mathematically wrong but produces correct class rankings in practice
- 比较 Multinomial, Bernoulli, and Gaussian 朴素贝叶斯 variants and select the right one for a given 特征 type
- 评估 朴素贝叶斯 against 逻辑回归 on high-dimensional sparse data and explain the 偏差-方差 tradeoff at work

## 问题

You need to classify text. Emails into spam or not-spam. Customer reviews into positive or negative. Support tickets into categories. You have thousands of 特征 (one per word) and limited 训练数据.

Most classifiers choke here. 逻辑回归 needs enough 样本 to estimate thousands of 权重 reliably. 决策树 划分 on one word at a time and overfit wildly. KNN in 10,000 dimensions is meaningless because every point is equally far from every other point.

朴素贝叶斯 handles this. It makes a mathematically wrong assumption (that every 特征 is independent of every other 特征 given the class), and it still outperforms "smarter" 模型 on text 分类, especially with small training sets. It trains in a single pass through the data. It scales to millions of 特征. It produces 概率 estimates (though often poorly calibrated due to the independence assumption).

Understanding why a wrong assumption leads to good 预测 teaches you something fundamental about 机器学习: the best 模型 is not the most correct one, it is the one with the best 偏差-方差 tradeoff for your data.

## 概念

### Bayes' Theorem (Quick Review)

Bayes' theorem flips conditional 概率:

```
P(class | features) = P(features | class) * P(class) / P(features)
```

We want `P(class | features)` -- the 概率 that a document belongs to a class given the words in it. We can compute this from:
- `P(features | class)` -- the likelihood of seeing these words in documents of this class
- `P(class)` -- the prior 概率 of the class (how common is spam in general?)
- `P(features)` -- the evidence, same for all classes, so we can ignore it when comparing

The class with the highest `P(class | features)` wins.

### The Naive Independence Assumption

Computing `P(features | class)` exactly requires estimating the joint 概率 of all 特征 together. With a vocabulary of 10,000 words, you would need to estimate a distribution over 2^10,000 possible combinations. Impossible.

The naive assumption: every 特征 is conditionally independent given the class.

```
P(w1, w2, ..., wn | class) = P(w1 | class) * P(w2 | class) * ... * P(wn | class)
```

Instead of one impossible joint distribution, you estimate n simple per-特征 distributions. Each one needs only a count.

This assumption is obviously wrong. The words "machine" and "learning" are not independent in any document. But the classifier does not need correct 概率 estimates. It needs correct rankings -- which class has the highest 概率. The independence assumption introduces systematic 误差, but those 误差 affect all classes similarly, so the ranking stays correct.

### 原因 It Still Works

Three reasons:

1. **Ranking over calibration.** 分类 only needs the top-ranked class to be correct. Even if P(spam) = 0.99999 when the true 概率 is 0.7, the classifier still picks spam correctly. We do not need correct 概率. We need the correct winner.

2. **High 偏差, low 方差.** The independence assumption is a strong prior. It constrains the 模型 heavily, which prevents 过拟合. With limited 训练数据, a 模型 that is slightly wrong but stable beats a 模型 that is theoretically right but wildly unstable. This is the 偏差-方差 tradeoff in action.

3. **特征 redundancy cancels out.** Correlated 特征 provide redundant evidence. The classifier double-counts this evidence, but it double-counts it for the correct class too. If "machine" and "learning" always appear together, both provide evidence for the "tech" class. NB counts them twice, but it counts them twice for the right class.

A fourth, practical reason: 朴素贝叶斯 is extremely fast. Training is a single pass through the data counting frequencies. 预测 is a matrix multiplication. You can train on a million documents in seconds. This speed means you can iterate faster, try more 特征 sets, and run more experiments than with slower 模型.

### The Math Step by Step

Let us trace through a concrete example. Suppose we have two classes: spam and not-spam. Our vocabulary has three words: "free", "money", "meeting".

训练数据:
- Spam emails mention "free" 80 times, "money" 60 times, "meeting" 10 times (150 total words)
- Not-spam emails mention "free" 5 times, "money" 10 times, "meeting" 100 times (115 total words)
- 40% of emails are spam, 60% are not-spam

With Laplace smoothing (alpha=1):

```
P(free | spam)    = (80 + 1) / (150 + 3) = 81/153 = 0.529
P(money | spam)   = (60 + 1) / (150 + 3) = 61/153 = 0.399
P(meeting | spam) = (10 + 1) / (150 + 3) = 11/153 = 0.072

P(free | not-spam)    = (5 + 1) / (115 + 3) = 6/118 = 0.051
P(money | not-spam)   = (10 + 1) / (115 + 3) = 11/118 = 0.093
P(meeting | not-spam) = (100 + 1) / (115 + 3) = 101/118 = 0.856
```

New email contains: "free" (2 times), "money" (1 time), "meeting" (0 times).

```
log P(spam | email) = log(0.4) + 2*log(0.529) + 1*log(0.399) + 0*log(0.072)
                    = -0.916 + 2*(-0.637) + (-0.919) + 0
                    = -3.109

log P(not-spam | email) = log(0.6) + 2*log(0.051) + 1*log(0.093) + 0*log(0.856)
                        = -0.511 + 2*(-2.976) + (-2.375) + 0
                        = -8.838
```

Spam wins by a large 间隔. The word "free" appearing twice is strong evidence for spam. Note that "meeting" not appearing contributes zero to both log sums (0 * log(P)) -- in Multinomial NB, absent words have no effect. It is Bernoulli NB that explicitly 模型 word absence.

### Three Variants

朴素贝叶斯 comes in three flavors. Each 模型 `P(feature | class)` differently.

#### Multinomial 朴素贝叶斯

模型 each 特征 as a count. Best for text data where 特征 are word frequencies or TF-IDF values.

```
P(word_i | class) = (count of word_i in class + alpha) / (total words in class + alpha * vocab_size)
```

The `alpha` is Laplace smoothing (explained below). This variant is the workhorse for text 分类.

#### Gaussian 朴素贝叶斯

模型 each 特征 as a normal distribution. Best for continuous 特征.

```
P(x_i | class) = (1 / sqrt(2 * pi * var)) * exp(-(x_i - mean)^2 / (2 * var))
```

Each class gets its own mean and 方差 per 特征. This works well when 特征 genuinely follow a bell curve within each class.

#### Bernoulli 朴素贝叶斯

模型 each 特征 as binary (present or absent). Best for short text or binary 特征 vectors.

```
P(word_i | class) = (docs in class containing word_i + alpha) / (total docs in class + 2 * alpha)
```

Unlike Multinomial, Bernoulli explicitly penalizes the absence of a word. If "free" typically appears in spam but is absent from this email, Bernoulli counts that as evidence against spam.

### When to Use Each Variant

| Variant | 特征 Type | Best For | Example |
|---------|-------------|----------|---------|
| Multinomial | Counts or frequencies | Text 分类, bag-of-words | Email spam, topic 分类 |
| Gaussian | Continuous values | Tabular data with normal-ish 特征 | Iris 分类, sensor data |
| Bernoulli | Binary (0/1) | Short text, binary 特征 vectors | SMS spam, presence/absence 特征 |

### Laplace Smoothing

What happens when a word appears in the 测试数据 but never appeared in the 训练数据 for a particular class?

Without smoothing: `P(word | class) = 0/N = 0`. One zero multiplied through the entire product makes `P(class | features) = 0`, regardless of all other evidence. A single unseen word destroys the entire 预测, no matter how much other evidence supports it.

Laplace smoothing adds a small count `alpha` (usually 1) to every 特征 count:

```
P(word_i | class) = (count(word_i, class) + alpha) / (total_words_in_class + alpha * vocab_size)
```

With alpha=1, every word gets at least a tiny 概率. The word "discombobulate" appearing in a test email no longer kills the spam 概率. The smoothing has a Bayesian interpretation: it is equivalent to placing a uniform Dirichlet prior on the word distributions.

Higher alpha means stronger smoothing (more uniform distributions). Lower alpha means the 模型 trusts the data more. Alpha is a 超参数 you tune.

The effect of alpha:

| Alpha | Effect | When to use |
|-------|--------|-------------|
| 0.001 | Almost no smoothing, trust the data | Very large 训练集, no unseen 特征 expected |
| 0.1 | Light smoothing | Large 训练集 |
| 1.0 | Standard Laplace smoothing | Default starting point |
| 10.0 | Heavy smoothing, flattens distributions | Very small 训练集, many unseen 特征 expected |

### Log-Space Computation

Multiplying hundreds of 概率 (each less than 1) causes floating-point underflow. The product becomes zero in floating point even though the true value is a very small positive number.

The solution: work in log space. Instead of multiplying 概率, add their logarithms:

```
log P(class | x1, x2, ..., xn) = log P(class) + sum_i log P(xi | class)
```

This turns the 预测 into a dot product:

```
log_scores = X @ log_feature_probs.T + log_class_priors
prediction = argmax(log_scores)
```

Matrix multiplication. That is why 朴素贝叶斯 预测 is so fast -- it is the same operation as a single-layer linear 模型.

### 朴素贝叶斯 vs 逻辑回归

Both are linear classifiers for text. The difference is in what they 模型.

| 方面 | 朴素贝叶斯 | 逻辑回归 |
|--------|------------|-------------------|
| Type | Generative (模型 P(X\|Y)) | Discriminative (模型 P(Y\|X)) |
| Training | Count frequencies | Optimize 损失函数 |
| Small data | Better (strong prior helps) | Worse (not enough to estimate 权重) |
| Large data | Worse (wrong assumption hurts) | Better (flexible boundary) |
| 特征 | Assumes independence | Handles correlations |
| Speed | Single pass, very fast | Iterative optimization |
| Calibration | Poor 概率 | Better 概率 |

Rule of thumb: start with 朴素贝叶斯. If you have enough data and NB plateaus, switch to 逻辑回归.

### 分类 流水线

```mermaid
flowchart LR
    A[Raw Text] --> B[Tokenize]
    B --> C[构建 Vocabulary]
    C --> D[Count Word Frequencies]
    D --> E[Apply Smoothing]
    E --> F[计算 Log 概率]
    F --> G[Predict: argmax P class given words]

    style A fill:#f9f,stroke:#333
    style G fill:#9f9,stroke:#333
```

In practice, we work in log space to avoid floating-point underflow. Instead of multiplying many small 概率, we add their logarithms:

```
log P(class | features) = log P(class) + sum_i log P(feature_i | class)
```

```figure
naive-bayes
```

## 动手构建

The code in `code/naive_bayes.py` implements both MultinomialNB and GaussianNB 从零实现.

### MultinomialNB

The from-scratch implementation:

1. **fit(X, y)**: For each class, count the frequency of each 特征. Add Laplace smoothing. 计算 log 概率. Store class priors (log of class frequencies).

2. **predict_log_proba(X)**: For each 样本, compute log P(class) + sum of log P(feature_i | class) for all classes. This is a matrix multiplication: X @ log_probs.T + log_priors.

3. **predict(X)**: Return the class with highest log 概率.

```python
class MultinomialNB:
    def __init__(self, alpha=1.0):
        self.alpha = alpha

    def fit(self, X, y):
        classes = np.unique(y)
        n_classes = len(classes)
        n_features = X.shape[1]

        self.classes_ = classes
        self.class_log_prior_ = np.zeros(n_classes)
        self.feature_log_prob_ = np.zeros((n_classes, n_features))

        for i, c in enumerate(classes):
            X_c = X[y == c]
            self.class_log_prior_[i] = np.log(X_c.shape[0] / X.shape[0])
            counts = X_c.sum(axis=0) + self.alpha
            self.feature_log_prob_[i] = np.log(counts / counts.sum())

        return self
```

The key insight: after fitting, 预测 is just matrix multiplication plus a 偏差. This is why 朴素贝叶斯 is so fast.

### GaussianNB

For continuous 特征, we estimate mean and 方差 per class per 特征:

```python
class GaussianNB:
    def __init__(self):
        pass

    def fit(self, X, y):
        classes = np.unique(y)
        self.classes_ = classes
        self.means_ = np.zeros((len(classes), X.shape[1]))
        self.vars_ = np.zeros((len(classes), X.shape[1]))
        self.priors_ = np.zeros(len(classes))

        for i, c in enumerate(classes):
            X_c = X[y == c]
            self.means_[i] = X_c.mean(axis=0)
            self.vars_[i] = X_c.var(axis=0) + 1e-9
            self.priors_[i] = X_c.shape[0] / X.shape[0]

        return self
```

预测 uses the Gaussian PDF per 特征, multiplied across 特征 (added in log space).

### Demo: Text 分类

The code generates synthetic bag-of-words data simulating two classes (tech articles vs sports articles). Each class has a different word frequency distribution. MultinomialNB classifies them using word counts.

The synthetic data works like this: we create 200 "words" (特征 columns). Words 0-39 have high frequency in tech articles and low in sports. Words 80-119 have high frequency in sports and low in tech. Words 40-79 are medium frequency in both. This creates a realistic scenario where some words are strong class indicators and others are noise.

### Demo: Continuous 特征

The code generates Iris-like data (3 classes, 4 特征, Gaussian clusters). GaussianNB classifies using per-class mean and 方差. Each class has a different center (mean vector) and different spread (方差), mimicking real-world data where measurements differ systematically between categories.

The code also demonstrates:
- **Smoothing comparison:** Training MultinomialNB with different alpha values to show the effect of smoothing strength on 准确率.
- **Training size experiment:** How NB 准确率 improves as 训练数据 grows from 20 to 1600 样本. NB reaches decent 准确率 even with very few 样本 -- this is its main advantage.
- **混淆矩阵:** Per-class 精确率, 召回率, and F1 score to show where NB makes mistakes.

### 预测 Speed

朴素贝叶斯 预测 is a matrix multiplication. For n 样本 with d 特征 and k classes:
- MultinomialNB: one matrix multiply (n x d) @ (d x k) = O(n * d * k)
- GaussianNB: n * k Gaussian PDF evaluations, each over d 特征 = O(n * d * k)

Both are linear in every dimension. 比较 this to KNN (which requires 距离 computation to all training points) or SVM with RBF 核函数 (which requires 核函数 evaluation against all 支持向量). NB is faster by orders of magnitude at 预测 time.

## 直接使用

With sklearn, both variants are one-liners:

```python
from sklearn.naive_bayes import GaussianNB, MultinomialNB

gnb = GaussianNB()
gnb.fit(X_train, y_train)
print(f"GaussianNB accuracy: {gnb.score(X_test, y_test):.3f}")

mnb = MultinomialNB(alpha=1.0)
mnb.fit(X_train_counts, y_train)
print(f"MultinomialNB accuracy: {mnb.score(X_test_counts, y_test):.3f}")
```

For text 分类 with sklearn:

```python
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline

text_clf = Pipeline([
    ("vectorizer", CountVectorizer()),
    ("classifier", MultinomialNB(alpha=1.0)),
])

text_clf.fit(train_texts, train_labels)
accuracy = text_clf.score(test_texts, test_labels)
```

The code in `naive_bayes.py` compares from-scratch implementations against sklearn on the same data to verify correctness.

### TF-IDF with 朴素贝叶斯

Raw word counts give every word equal 权重 per occurrence. But common words like "the" and "is" appear frequently in every class -- they carry no information. TF-IDF (术语 Frequency - Inverse Document Frequency) downweights common words and upweights rare, discriminative words.

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline

text_clf = Pipeline([
    ("tfidf", TfidfVectorizer()),
    ("classifier", MultinomialNB(alpha=0.1)),
])
```

TF-IDF values are non-negative, so they work with MultinomialNB. The combination of TF-IDF + MultinomialNB is one of the strongest baselines for text 分类. It frequently beats more complex 模型 on 数据集 with fewer than 10,000 training 样本.

### BernoulliNB for Short Text

For short text (tweets, SMS, chat messages), BernoulliNB can outperform MultinomialNB. Short texts have low word counts, so the frequency information that MultinomialNB relies on is noisy. BernoulliNB only cares about presence or absence, which is more reliable with short text.

```python
from sklearn.naive_bayes import BernoulliNB
from sklearn.feature_extraction.text import CountVectorizer

text_clf = Pipeline([
    ("vectorizer", CountVectorizer(binary=True)),
    ("classifier", BernoulliNB(alpha=1.0)),
])
```

The `binary=True` flag in CountVectorizer converts all counts to 0/1. Without it, BernoulliNB still works but is seeing counts that it was not designed for.

### Calibrating NB 概率

NB 概率 are poorly calibrated. When NB says P(spam) = 0.95, the true 概率 might be 0.7. If you need reliable 概率 estimates (for example, to set a 阈值 or to combine with other 模型), use sklearn's CalibratedClassifierCV:

```python
from sklearn.calibration import CalibratedClassifierCV

calibrated_nb = CalibratedClassifierCV(MultinomialNB(), cv=5, method="sigmoid")
calibrated_nb.fit(X_train, y_train)
proba = calibrated_nb.predict_proba(X_test)
```

This fits a 逻辑回归 on top of NB's raw scores using 交叉验证. The resulting 概率 are much closer to the true class frequencies.

### Common Gotchas

1. **Negative 特征 values.** MultinomialNB requires non-negative 特征. If you have negative values (like TF-IDF with certain settings or standardized 特征), use GaussianNB instead, or shift the 特征 to be positive.

2. **Zero 方差 特征.** GaussianNB divides by 方差. If a 特征 has zero 方差 for a class (all values identical), the 概率 computation breaks. The code adds a small smoothing term (1e-9) to all variances to prevent this.

3. **Class imbalance.** If 99% of emails are not-spam, the prior P(not-spam) = 0.99 is so strong that it overwhelms the likelihood evidence. You can set class priors manually or use class_prior 参数 in sklearn.

4. **特征 scaling.** MultinomialNB does not need scaling (it works on counts). GaussianNB does not need scaling either (it estimates per-特征 statistics). This is an advantage over 逻辑回归 and SVM, which are sensitive to 特征 scales.

## 交付成果

本课产出：
- `outputs/skill-naive-bayes-chooser.md` -- a decision skill for picking the right NB variant
- `code/naive_bayes.py` -- MultinomialNB and GaussianNB 从零实现, with sklearn comparison

### When 朴素贝叶斯 Fails

NB fails when the independence assumption causes incorrect rankings (not just incorrect 概率). This happens when:

1. **Strong 特征 interactions.** If the class depends on the combination of two 特征 but not either alone (XOR-like patterns), NB will miss it entirely. Each 特征 alone provides no evidence, and NB cannot combine them nonlinearly.

2. **Highly correlated 特征 with opposing evidence.** If 特征 A says "spam" and 特征 B says "not-spam", but A and B are perfectly correlated (they always agree in reality), NB will see conflicting evidence where there is none.

3. **Very large training sets.** With enough data, discriminative 模型 like 逻辑回归 learn the true 决策边界 and outperform NB. The independence assumption that helped with small data now holds the 模型 back.

In practice, these failure modes are rare for text 分类. Text 特征 are numerous, individually weak, and the independence assumption's 误差 tend to cancel out. For tabular data with few strongly correlated 特征, consider 逻辑回归 or 树-based 模型 first.

## 练习

1. **Smoothing experiment.** Train MultinomialNB on text data with alpha values of 0.01, 0.1, 1.0, 10.0, and 100.0. Plot 准确率 vs alpha. Where does performance peak? 原因 does very high alpha hurt?

2. **特征 independence test.** Take a real text 数据集. Pick two words that are obviously correlated ("machine" and "learning"). 计算 P(word1 | class) * P(word2 | class) and compare to P(word1 AND word2 | class). How wrong is the independence assumption? Does it affect 分类 准确率?

3. **Bernoulli implementation.** Extend the code with a BernoulliNB class. Convert bag-of-words to binary (present/absent) and compare 准确率 against MultinomialNB on text data. When does Bernoulli win?

4. **NB vs 逻辑回归.** Train both on text data. Start with 100 training 样本 and increase to 10,000. Plot 准确率 vs 训练集 size for both. At what point does 逻辑回归 overtake 朴素贝叶斯?

5. **Spam filter.** 构建 a complete spam classifier: tokenize raw email text, build vocabulary, create bag-of-words 特征, train MultinomialNB, evaluate with 精确率 and 召回率 (not just 准确率 -- why?).

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|----------------|----------------------|
| 朴素贝叶斯 | "Simple probabilistic classifier" | A classifier that applies Bayes' theorem with the assumption that 特征 are conditionally independent given the class |
| Conditional independence | "特征 don't affect each other" | P(A, B \| C) = P(A \| C) * P(B \| C) -- knowing B tells you nothing new about A once you know C |
| Laplace smoothing | "Add-one smoothing" | Adding a small count to every 特征 to prevent zero 概率 from dominating the 预测 |
| Prior | "What you believed before seeing data" | P(class) -- the 概率 of each class before observing any 特征 |
| Likelihood | "How well the data fits" | P(特征 \| class) -- the 概率 of observing these 特征 if the class is known |
| Posterior | "What you believe after seeing data" | P(class \| 特征) -- the updated 概率 of the class after observing the 特征 |
| Generative 模型 | "模型 how data is generated" | A 模型 that learns P(X \| Y) and P(Y), then uses Bayes' theorem to get P(Y \| X) |
| Discriminative 模型 | "模型 the 决策边界" | A 模型 that directly learns P(Y \| X) without modeling how X is generated |
| Log 概率 | "Avoid underflow" | Working with log P instead of P to prevent the product of many small numbers from becoming zero in floating point |

## 延伸阅读

- [scikit-learn Naive Bayes docs](https://scikit-learn.org/stable/modules/naive_bayes.html) -- all three variants with mathematical details
- [McCallum and Nigam, A Comparison of Event Models for Naive Bayes Text Classification (1998)](https://www.cs.cmu.edu/~knigam/papers/multinomial-aaaiws98.pdf) -- the classic comparison of Multinomial vs Bernoulli for text
- [Rennie et al., Tackling the Poor Assumptions of Naive Bayes Text Classifiers (2003)](https://people.csail.mit.edu/jrennie/papers/icml03-nb.pdf) -- improvements to NB for text
- [Ng and Jordan, On Discriminative vs. Generative Classifiers (2001)](https://ai.stanford.edu/~ang/papers/nips01-discriminativegenerative.pdf) -- proves NB converges faster than LR with less data
