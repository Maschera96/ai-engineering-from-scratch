---
name: skill-evaluation-zh
description: Evaluation strategy checklist for 分类 and 回归 模型
version: 1.0.0
phase: 2
lesson: 9
tags: [evaluation, metrics, cross-validation, model-selection]
---

# 模型 Evaluation Strategy

A checklist for correctly evaluating any ML 模型. Follow this sequence to avoid the most common evaluation mistakes.

## Step 1: 划分 the data correctly

- 划分 before any preprocessing (scaling, imputation, encoding)
- Use stratified 划分 for 分类 tasks
- Reserve a 测试集 that you touch exactly once at the end
- For small 数据集, use 5-fold or 10-fold 交叉验证 instead of a single 划分
- For 时间序列, use time-based 划分 (never shuffle)

## Step 2: Pick the right 指标

### 分类

| Situation | Use this 指标 | 原因 |
|-----------|----------------|-----|
| Balanced classes, simple comparison | 准确率 | Easy to interpret, meaningful when classes are equal |
| False positives are costly (spam filter, fraud alerts) | 精确率 | Measures how many flagged items are actually positive |
| False negatives are costly (cancer screening, security) | 召回率 | Measures how many actual positives you catch |
| Need to balance 精确率 and 召回率 | F1 Score | Harmonic mean, punishes extreme imbalance |
| Comparing 模型 across thresholds | AUC-ROC | 阈值-independent ranking quality |
| 不平衡数据 | F1, AUC-ROC, or PR-AUC | 准确率 is misleading with imbalanced classes |

### 回归

| Situation | Use this 指标 | 原因 |
|-----------|----------------|-----|
| Standard 回归, outliers acceptable | RMSE | Same units as 目标, penalizes large 误差 |
| Outlier-robust evaluation | MAE | Treats all 误差 equally, not dominated by outliers |
| Comparing 模型 on different scales | R-squared | Normalized 0-1 scale (fraction of 方差 explained) |
| Business requires dollar amounts | MAE or RMSE | Directly interpretable as 误差 magnitude |

## Step 3: Establish baselines

Before evaluating your 模型, compute 基线 performance:
- 分类: 多数类 predictor (always predict the most common class)
- 回归: always predict the mean of the training 目标
- Any 模型 that cannot beat these baselines is not learning

## Step 4: Cross-validate

- Use K-fold (K=5 or K=10) for stable estimates
- Use stratified K-fold for 分类
- Report mean and standard deviation across folds
- A 模型 with mean=0.85 and std=0.02 is more trustworthy than mean=0.87 and std=0.10

## Step 5: 比较 模型 statistically

- Do not pick the 模型 with the highest average score without checking significance
- Use a paired t-test across 交叉验证 folds
- If |t| < 2.78 (for K=5, df=4, p<0.05), the difference may be due to chance
- Consider the simpler 模型 when performance differences are not significant

## Step 6: Check for common mistakes

- Data leakage: did any 测试数据 information flow into training? (scaling before splitting, 目标-derived 特征)
- Class imbalance: is 准确率 hiding poor minority-class performance?
- 过拟合: is the gap between training and validation performance large?
- Too many evaluations: have you looked at the 测试集 more than once?

## Step 7: Report final performance

- Train on train + validation combined
- 评估 on the held-out 测试集 exactly once
- Report the chosen 指标 with confidence intervals if possible
- State the 基线 comparison (how much better than random/mean)
