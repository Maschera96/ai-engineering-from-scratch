---
name: skill-probability-reasoning-zh
description: Choose the right 概率 分布 for a given ML problem
version: 1.0.0
phase: 1
lesson: 6
tags: [probability, distributions, modeling]
---

# 概率 分布 Selection

How to pick the right 分布 when modeling 数据, designing 损失 函数, 或 setting priors.

## Decision Checklist

1. Is the outcome discrete (categories, counts) 或 continuous (measurements, scores)?
2. Is the outcome bounded (e.g., [0, 1]) 或 unbounded?
3. How many possible outcomes are there? Two? k? Infinite?
4. Is the 数据 symmetric 或 skewed?
5. Are events independent 或 correlated?
6. Are you modeling a rate, a count, a proportion, 或 a measurement?

## 分布 decision tree

```
Is the variable discrete?
  Yes --> Only 2 outcomes? --> Bernoulli (p)
     |    k outcomes, one trial? --> Categorical (p1...pk)
     |    k outcomes, n trials? --> Multinomial (n, p1...pk)
     |    Count of successes in n trials? --> Binomial (n, p)
     |    Count of events per interval? --> Poisson (lambda)
     |    Count of trials until first success? --> Geometric (p)
     |    Count of trials until r successes? --> Negative Binomial (r, p)
  No --> Symmetric, bell-shaped? --> Normal (mu, sigma)
     |   Positive values, right-skewed? --> Log-normal or Exponential
     |   Bounded in [0, 1]? --> Beta (alpha, beta)
     |   Positive values, flexible shape? --> Gamma (alpha, beta)
     |   Time between events? --> Exponential (lambda)
     |   Heavy tails needed? --> Student's t (nu) or Cauchy
     |   Multivariate, bell-shaped? --> Multivariate Normal
     |   On a simplex (sums to 1)? --> Dirichlet (alpha)
```

## Mapping real-world ML scenarios to 分布

| Scenario | 分布 | Parameters |
|---|---|---|
| Binary 分类 输出 | Bernoulli | p = sigmoid(logit) |
| Multi-class 分类 输出 | Categorical | p = softmax(logits) |
| Token prediction in language 模型 | Categorical over vocab | p from softmax |
| Pixel intensity (normalized) | Beta 或 Uniform [0, 1] | Depends on image stats |
| Word count in a document | Poisson | lambda = avg word count |
| 时间 between user requests | Exponential | lambda = request rate |
| Measurement 误差 | Normal | mu = 0, sigma from 数据 |
| Weight initialization | Normal 或 Uniform | Kaiming/Xavier rules |
| VAE latent 空间 先验 | St和ard Normal | mu = 0, sigma = 1 |
| 贝叶斯 先验 on proportions | Beta | alpha, beta from belief |
| 贝叶斯 先验 on category weights | Dirichlet | alpha 向量 |
| Noise in 回归 targets | Normal | mu = 0, sigma estimated |
| Outlier-robust 回归 | Student's t | low degrees of freedom |
| Duration/lifetime modeling | Weibull 或 Gamma | 形状 和 scale |
| Topic 分布 per document (LDA) | Dirichlet | alpha < 1 for sparse |

## When 分布 go wrong

- Using Normal when 数据 has a hard lower bound (e.g., prices, 距离). The normal assigns nonzero 概率 to 负 values. Use log-normal 或 gamma instead.
- Using Poisson when the 方差 differs from the 均值. Poisson assumes 均值 = 方差. If 方差 > 均值, use 负 binomial.
- Using Bernoulli for multi-class problems. Bernoulli is strictly binary. Use categorical for k > 2.
- Assuming independence when observations are correlated. 时间 series, spatial 数据, 和 grouped 数据 violate independence. Use autoregressive 或 hierarchical 模型.

## Common mistakes

- Confusing PDF values 与 probabilities. A PDF can exceed 1. 概率 comes from integrating the PDF over an interval.
- Forgetting that softmax 输出 are categorical probabilities, not independent Bernoulli probabilities. They sum to 1 by construction.
- Using a uniform 先验 when you have domain knowledge. Informative priors reduce 方差 without biasing the result if chosen well.
- Treating log-probabilities as probabilities. Log-probs are always 负 (or zero). They do not sum to 1.

## Quick reference: 分布 properties

| 分布 | Support | Mean | 方差 | Key property |
|---|---|---|---|---|
| Bernoulli(p) | {0, 1} | p | p(1-p) | Simplest discrete |
| Binomial(n, p) | {0..n} | np | np(1-p) | Sum of n Bernoulli |
| Poisson(lam) | {0, 1, 2, ...} | lam | lam | Mean = 方差 |
| Normal(mu, s^2) | (-inf, inf) | mu | s^2 | Max 熵 for given 均值/var |
| Exponential(lam) | [0, inf) | 1/lam | 1/lam^2 | Memoryless |
| Beta(a, b) | [0, 1] | a/(a+b) | ab/((a+b)^2(a+b+1)) | Conjugate to Binomial |
| Gamma(a, b) | (0, inf) | a/b | a/b^2 | Conjugate to Poisson |
| Dirichlet(alpha) | Simplex | alpha_i/sum | (see formula) | Conjugate to Categorical |
