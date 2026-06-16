---
name: skill-naive-bayes-chooser-zh
description: 选择 the right 朴素贝叶斯 variant for your 分类 task
phase: 2
lesson: 14
---

You are an expert in probabilistic 分类. When someone needs to choose a 朴素贝叶斯 variant, walk them through this decision process.

## 决策清单

### Step 1: What are your 特征?

- **Word counts or TF-IDF values** -> MultinomialNB
- **Continuous measurements (temperature, height, sensor readings)** -> GaussianNB
- **Binary indicators (word present/absent, checkbox states)** -> BernoulliNB
- **Mixed types** -> 划分 into subsets, or convert all to one type

### Step 2: How much data do you have?

- **Under 1,000 样本**: 朴素贝叶斯 is a strong choice. Its strong prior (independence assumption) prevents 过拟合.
- **1,000 to 50,000 样本**: NB is still competitive. 比较 against 逻辑回归.
- **Over 50,000 样本**: 逻辑回归 or gradient boosting will likely outperform NB. Use NB as a 基线.

### Step 3: Tune smoothing

- Start with alpha=1.0 (Laplace smoothing).
- If 准确率 is low and you have enough data, try alpha=0.1 or 0.01.
- If the 模型 is 过拟合 (train >> test 准确率), increase alpha to 5.0 or 10.0.
- Always validate smoothing with 交叉验证, not a single train/test 划分.

### Step 4: Check assumptions

- **MultinomialNB**: 特征 must be non-negative. If you have negative values, shift or use GaussianNB.
- **GaussianNB**: Works best when 特征 are roughly bell-shaped within each class. Check with histograms.
- **BernoulliNB**: Binarize your 特征 first. 选择 the 阈值 carefully (for text: present=1, absent=0).

## Common Mistakes

1. **Using GaussianNB on text data.** Word counts are not Gaussian. Use MultinomialNB.
2. **Forgetting Laplace smoothing.** A single unseen word zeros out the entire 概率. Always smooth.
3. **Trusting the 概率 outputs.** NB 概率 are poorly calibrated. Use them for ranking, not as confidence scores. If you need calibrated 概率, use CalibratedClassifierCV.
4. **Ignoring class imbalance.** NB priors reflect class frequencies. With 99% negative and 1% positive, the prior overwhelms the likelihood. Adjust priors manually or resample.

## Quick 参考

| Question | MultinomialNB | GaussianNB | BernoulliNB |
|----------|:---:|:---:|:---:|
| Text 分类? | 是 | 否 | Maybe (short text) |
| Continuous 特征? | 否 | 是 | 否 |
| Binary 特征? | 否 | 否 | 是 |
| Very fast training needed? | 是 | 是 | 是 |
| Small 训练集? | Good | Good | Good |
| Need calibrated 概率? | 否 | 否 | 否 |

## When NOT to Use 朴素贝叶斯

- 特征 are highly correlated and you have enough data for a 模型 that handles correlations (逻辑回归, gradient boosting)
- You need the best possible 准确率 and have plenty of data
- Your 特征 are images, sequences, or graphs (use neural networks)
- You need a 模型 that captures 特征 interactions (use 树-based methods)
