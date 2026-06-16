---
name: skill-classification-baseline-zh
description: Establish a strong 分类 基线 before reaching for complex 模型
version: 1.0.0
phase: 2
lesson: 3
tags: [classification, logistic-regression, baseline, preprocessing]
---

# 分类 基线 Guide

Before trying complex 模型, establish a 基线 with 逻辑回归. It trains in seconds, produces 概率, and is fully interpretable. A surprising number of real-world problems never need anything fancier.

## 决策清单

1. Is the 决策边界 likely linear?
   - 是: 逻辑回归 will probably be sufficient
   - 否: you still want it as a 基线 to measure improvement

2. How many 特征 do you have?
   - Under 50: standard 逻辑回归 works fine
   - 50 to 10,000: add L2 正则化 (Ridge)
   - Over 10,000 (e.g., TF-IDF text 特征): use L1 正则化 (Lasso) or LinearSVC

3. Is the 数据集 imbalanced?
   - Under 5:1 ratio: probably fine without adjustment
   - 5:1 to 50:1: use `class_weight="balanced"` in sklearn
   - Over 50:1: combine class weighting with appropriate 指标 (精确率, 召回率, or F1)

4. Are 特征 on different scales?
   - Always standardize before 逻辑回归. It uses gradient-based optimization, and unscaled 特征 slow convergence or distort the 决策边界.

5. Are there missing values?
   - Impute before fitting. 逻辑回归 cannot handle NaNs.
   - Use median imputation for numeric columns, mode for categorical.

## When 逻辑回归 is good enough

- Binary 分类 with mostly linear 特征 relationships
- You need 概率 outputs (not just class 标签)
- Interpretability is required (coefficients indicate 特征 importance direction and relative magnitude after standardization)
- 训练数据 is small (hundreds to low thousands of 样本)
- You need a fast 模型 for real-time serving (single dot product at inference)
- Regulatory or compliance requirements demand explainability

## When to upgrade

- 准确率 plateaus well below the 目标 and you have tried 特征工程
- The relationship between 特征 and 目标 is clearly nonlinear (check 残差 plots)
- You have large tabular data (10k+ rows): try gradient boosting (XGBoost or LightGBM)
- 特征 have complex interactions that polynomial 特征 cannot capture
- You have image, text, or sequential data: 逻辑回归 on raw inputs will not work

## Preprocessing steps for a 分类 基线

1. **Train/test 划分** first, before any preprocessing. This prevents data leakage.
2. **Handle missing values**: median impute numeric, mode impute categorical.
3. **Encode categoricals**: one-hot for low cardinality (under 10 values), 目标 encoding for higher. Fit 目标 encoding only on training folds (use out-of-fold encoding to prevent leakage).
4. **Scale numerics**: StandardScaler (zero mean, unit 方差). Fit on train, transform both.
5. **Fit 逻辑回归** with `C=1.0` (default 正则化).
6. **评估**: 混淆矩阵, 精确率, 召回率, F1. Not just 准确率.
7. **Tune 阈值**: default 0.5 is rarely optimal. Sweep 0.1 to 0.9 and pick the 阈值 that matches your 精确率/召回率 priority.

## 常见错误

- Evaluating only 准确率 on 不平衡数据 (a 模型 predicting the 多数类 scores high but is useless)
- Forgetting to scale 特征 (逻辑回归 with unscaled 特征 trains slowly and converges to a worse solution)
- Using the 测试集 to tune the decision 阈值 (use validation or 交叉验证)
- Skipping the 基线 and jumping straight to XGBoost (you lose interpretability and have no reference point)
- Not checking for multicollinearity (highly correlated 特征 inflate coefficient 方差)

## 速查表

| Scenario | 模型 | 正则化 | Key setting |
|----------|-------|---------------|-------------|
| Few 特征, interpretable | LogisticRegression | L2 (default) | C=1.0 |
| Many 特征, some irrelevant | LogisticRegression | L1 | penalty="l1", solver="saga" |
| High-dim sparse (text) | SGDClassifier | L1 or ElasticNet | loss="log_loss" |
| Imbalanced classes | LogisticRegression | L2 | class_weight="balanced" |
| Need 概率 | LogisticRegression | L2 | predict_proba() |
| Need class 标签 only | LinearSVC | L2 | Faster than LR for large data |
