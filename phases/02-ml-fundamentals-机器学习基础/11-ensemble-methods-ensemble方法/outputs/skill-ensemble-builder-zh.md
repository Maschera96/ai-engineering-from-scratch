---
name: skill-ensemble-builder-zh
description: 选择 the right 集成 method and configure it for your problem
version: 1.0.0
phase: 2
lesson: 11
tags: [ensemble, bagging, boosting, random-forest, xgboost, stacking]
---

# 集成 Method Selection Guide

Ensembles combine multiple 模型 to produce better 预测 than any single 模型. The question is always: which kind of 集成, and when?

## 决策清单

1. What is the main problem with your current 模型?
   - High 方差 (过拟合): use bagging (随机森林)
   - High 偏差 (欠拟合): use boosting (Gradient boosting, XGBoost)
   - Both, or you want maximum 准确率: use stacking

2. How much data do you have?
   - Under 1,000 rows: 随机森林 (robust, hard to misconfigure)
   - 1,000 to 100,000: XGBoost or LightGBM (best overall for tabular)
   - Over 100,000: LightGBM (fastest gradient boosting, handles large data well)

3. How much tuning time can you invest?
   - Minimal: 随机森林 with defaults (almost always works)
   - Moderate: XGBoost with learning_rate=0.1, tune n_estimators with early stopping
   - Maximum: LightGBM or XGBoost with Bayesian 超参数 search

4. Do you need interpretability?
   - 是: single 决策树 or small 随机森林 with 特征 importance
   - Partial: gradient boosting with SHAP values
   - 否: stacking or deep ensembles

5. Is the data noisy with many outliers?
   - 是: 随机森林 (bagging is robust to noise)
   - 否: gradient boosting (can push 准确率 further on clean data)

## When to use each method

**随机森林 (bagging)**: your safe first choice. Trains many 树 on bootstrap 样本 and averages. Reduces 方差 without increasing 偏差. Nearly impossible to overfit on moderate data. Minimal tuning needed: set n_estimators=100-500 and leave defaults.

**AdaBoost**: sequential boosting with 样本 reweighting. Works well with simple base learners (decision stumps). Sensitive to outliers and noisy 标签 because it upweights misclassified points. Largely replaced by gradient boosting in practice.

**Gradient boosting**: fits each new 树 to the 残差 of the 集成 so far. Reduces 偏差. The most powerful method for tabular data. Requires tuning: learning_rate, n_estimators, max_depth, min_child_weight, subsample.

**XGBoost**: gradient boosting with 正则化, second-order optimization, and systems-level speedups. Handles missing values natively. The default for Kaggle competitions and 生产环境 ML on tabular data.

**LightGBM**: gradient boosting with 叶节点-wise growth (instead of level-wise). Faster than XGBoost on large 数据集. Uses histogram-based 划分. Best for 数据集 over 50k rows.

**CatBoost**: gradient boosting with native categorical 特征 handling. 否 need to one-hot encode. Good when you have many categorical 特征.

**stacking**: trains a meta-learner on the 预测 of multiple diverse base 模型. Use when you need the absolute best 准确率 and have compute to spare. Always generate base 模型 预测 via 交叉验证 to avoid leakage.

**Voting**: simplest 集成. Hard voting (多数类) or soft voting (average 概率). Quick way to combine 2-3 diverse 模型 without a meta-learner.

## 常见错误

- Using gradient boosting without early stopping (it will overfit if you let it run too many rounds)
- Setting learning_rate too high (above 0.3 usually causes instability)
- Not tuning max_depth for gradient boosting (default of unlimited or very deep 树 overfit)
- stacking with 模型 that are all the same type (diversity is the point of stacking)
- Using AdaBoost on noisy data (outliers get higher and higher 权重 each round)
- Expecting 随机森林 to fix 欠拟合 (it reduces 方差, not 偏差)

## Tuning priorities by method

**随机森林:**
1. n_estimators: 100-500 (more is rarely worse, just slower)
2. max_depth: None (let 树 grow fully) or cap at 10-20 for speed
3. max_features: "sqrt" for 分类, "log2" or n/3 for 回归

**XGBoost / LightGBM:**
1. learning_rate: 0.01-0.3 (lower is better if you have compute for more 树)
2. n_estimators: use early stopping on a 验证集 instead of guessing
3. max_depth: 3-8 (start with 6)
4. min_child_weight / min_data_in_leaf: 1-20 (higher prevents 过拟合)
5. subsample: 0.7-1.0
6. colsample_bytree: 0.7-1.0
7. reg_alpha (L1) and reg_lambda (L2): 0-10

## 速查表

| Method | Reduces | Speed | Tuning effort | Best for |
|--------|---------|-------|--------------|----------|
| 随机森林 | 方差 | Fast | Low | Noisy data, quick 基线 |
| AdaBoost | 偏差 | Fast | Low | Simple base learners, clean data |
| Gradient boosting | 偏差 | Medium | High | Tabular data, competitions |
| XGBoost | Both | Fast | High | 生产环境 tabular ML |
| LightGBM | Both | Fastest | High | Large 数据集 (50k+ rows) |
| CatBoost | Both | Medium | Medium | Many categorical 特征 |
| stacking | Both | Slow | High | Maximum 准确率, diverse 模型 |
| Voting | 方差 | Fast | None | Quick combination of 2-3 模型 |
