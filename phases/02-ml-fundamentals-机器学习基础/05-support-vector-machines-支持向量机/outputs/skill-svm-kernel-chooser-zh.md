---
name: skill-svm-kernel-chooser-zh
description: 选择 the right SVM 核函数 and tune C and gamma for your problem
version: 1.0.0
phase: 2
lesson: 5
tags: [svm, kernel, classification, hyperparameter-tuning]
---

# SVM 核函数 Selection Guide

SVMs are defined by two choices: the 核函数 (which determines the shape of the 决策边界) and the 正则化 参数 (which control the tradeoff between 间隔 width and 分类 误差). Getting these right is the difference between a useless 模型 and a strong one.

## 决策清单

1. Is the data linearly separable (or close to it)?
   - 是: use linear 核函数. It is faster and more interpretable.
   - 否: go to step 2.

2. How many 特征 vs 样本?
   - 特征 >> 样本 (e.g., text with TF-IDF): use linear 核函数. High-dimensional data is often linearly separable. RBF adds complexity for no gain.
   - 样本 >> 特征 (e.g., tabular data with 10-50 特征): RBF 核函数 is the default choice.

3. Is the 决策边界 expected to be smooth?
   - Smooth, continuous boundary: RBF 核函数
   - Polynomial-shaped boundary: polynomial 核函数 (start with degree 2 or 3)
   - Domain knowledge suggests specific interaction terms: polynomial 核函数 with matching degree

4. How large is the 数据集?
   - Under 10,000 样本: any 核函数 works, RBF is the safe default
   - 10,000 to 100,000: linear 核函数 or LinearSVC (primal formulation, O(n) per epoch)
   - Over 100,000: do not use 核函数 SVM. Switch to linear SVM, gradient boosting, or neural networks.

5. Did you scale the 特征?
   - SVMs require 特征 scaling. Always standardize (zero mean, unit 方差) before fitting. Unscaled 特征 distort the 间隔 geometry.

## 核函数 selection flowchart

```
Start
  |
  v
Features > 1000 or features >> samples?
  Yes --> Linear kernel (LinearSVC for speed)
  No  --> Dataset < 10k samples?
            Yes --> Try RBF first (best general-purpose kernel)
            No  --> Linear kernel (kernel SVMs are O(n^2) to O(n^3))
```

If RBF does not work well, try polynomial degree 2-3. If that fails, the problem may not be suited to SVMs.

## Tuning C (正则化)

C controls the penalty for misclassifications. It is inversely related to 正则化 strength.

| C value | Effect | When to use |
|---------|--------|-------------|
| 0.001 - 0.01 | Wide 间隔, many violations allowed | Noisy data, want 泛化 |
| 0.1 - 1.0 | Balanced | Good starting range |
| 10 - 1000 | Narrow 间隔, few violations | Clean data, need high 准确率 |

Tuning strategy:
- Start with C=1.0
- Search on a log scale: [0.001, 0.01, 0.1, 1, 10, 100, 1000]
- Use 交叉验证 to pick the best value
- If best C is at the edge of your range, extend the range in that direction

## Tuning gamma (RBF 核函数)

Gamma controls how far the influence of a single training point reaches. It defines the width of the Gaussian.

| gamma value | Effect | When to use |
|-------------|--------|-------------|
| Small (0.001) | Each point influences a large area. Smooth, simple boundary | 欠拟合 or few 特征 |
| Medium (auto: 1/n_features) | sklearn default. Reasonable starting point | General use |
| Large (10+) | Each point influences only nearby points. Complex, wiggly boundary | Risk of 过拟合 |

Tuning strategy:
- Start with gamma="scale" (1 / (n_features * X.var()), the sklearn default)
- Search on a log scale: [0.001, 0.01, 0.1, 1, 10]
- Low gamma + high C tends to overfit
- High gamma + low C tends to underfit

## Joint C and gamma tuning

C and gamma interact. Always tune them together, not independently.

Recommended approach:
1. Coarse grid search: C in [0.01, 0.1, 1, 10, 100], gamma in [0.001, 0.01, 0.1, 1, 10] (25 combos)
2. Find the best region
3. Fine grid search around the best region (e.g., C in [5, 10, 20, 50], gamma in [0.05, 0.1, 0.2])
4. Use 5-fold 交叉验证 throughout

## 常见错误

- Using RBF 核函数 on high-dimensional sparse data (linear is better and 100x faster)
- Forgetting to scale 特征 (the single most common SVM mistake)
- Setting C too high on noisy data (memorizes noise instead of learning the boundary)
- Using 核函数 SVM on 数据集 over 50k 样本 (training time is prohibitive)
- Not tuning C and gamma together (they compensate for each other)
- Defaulting to polynomial degree 5+ (overfits aggressively, try 2 or 3 first)

## 速查表

| 核函数 | When to use | Key 参数 | Training complexity |
|--------|------------|----------------|-------------------|
| Linear | Text/TF-IDF, many 特征, large data | C only | O(n) per epoch |
| RBF | General-purpose, under 10k 样本 | C, gamma | O(n^2) to O(n^3) |
| Polynomial | Known polynomial relationships | C, degree, coef0 | O(n^2) to O(n^3) |
| Sigmoid | Rarely useful (equivalent to two-layer neural net) | C, gamma, coef0 | O(n^2) to O(n^3) |
