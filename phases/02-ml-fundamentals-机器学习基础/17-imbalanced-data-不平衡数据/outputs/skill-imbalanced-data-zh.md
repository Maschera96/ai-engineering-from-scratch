---
name: skill-imbalanced-data-zh
description: Decision checklist for handling imbalanced 分类 problems
version: 1.0.0
phase: 2
lesson: 17
tags: [imbalanced-data, smote, class-weights, threshold-tuning, evaluation]
---

# 不平衡数据 Strategy

A decision checklist for handling imbalanced 分类. Follow this sequence to pick the right approach for your problem.

## Step 1: Measure the imbalance

- Count 样本 per class
- 计算 the imbalance ratio (majority / minority)
- Mild: ratio < 3:1 (e.g., 70/30)
- Moderate: ratio 3:1 to 20:1 (e.g., 95/5)
- Severe: ratio > 20:1 (e.g., 99/1)

## Step 2: Pick the right 指标

Prefer 精确率/召回率/F1 over 准确率 for imbalanced 数据集. 选择 based on your problem:

| Situation | Primary 指标 | Secondary 指标 |
|-----------|---------------|-----------------|
| Missing positives is very costly (fraud, disease) | 召回率 | F2 score |
| False alarms are costly (spam filter, recommendations) | 精确率 | F0.5 score |
| Both matter roughly equally | F1 score | MCC |
| Need a single ranking 指标 | AUPRC | AUC-ROC |
| Need to compare across 数据集 | MCC | AUPRC |

## Step 3: 选择 a rebalancing strategy

### By imbalance severity

| Imbalance | First Try | Second Try | Avoid |
|-----------|-----------|------------|-------|
| Mild (< 3:1) | Class 权重 | 阈值 tuning | Oversampling (unnecessary) |
| Moderate (3:1 to 20:1) | SMOTE + class 权重 | 阈值 tuning on top | Undersampling (too much data loss) |
| Severe (> 20:1) | SMOTE + class 权重 + 阈值 | 集成 with balanced bagging | Undersampling alone |

### By 数据集 size

| 数据集 Size | Preferred Strategy | Reason |
|-------------|-------------------|--------|
| < 1,000 样本 | Oversampling or SMOTE | Cannot afford to lose majority data |
| 1,000 - 10,000 | SMOTE + 阈值 tuning | Enough minority 样本 for k-NN |
| > 10,000 | Class 权重 or undersampling | Fast, sufficient minority data |

## Step 4: Apply the technique

### Class 权重 (always try first)
- In sklearn: `class_weight='balanced'`
- 否 data modification needed
- Works with any loss-based 模型
- Equivalent to oversampling in expectation

### SMOTE
- Apply only to 训练数据 (never test/validation)
- Use k=5 neighbors (default)
- Combine with class 权重 for best results
- Watch for noisy synthetic points near the boundary

### 阈值调优
- Train 模型, get predicted 概率 on 验证集
- Sweep thresholds from 0.05 to 0.95
- Pick 阈值 maximizing your chosen 指标
- Always tune on validation data, never 测试数据

## Step 5: Validate properly

- Use stratified 交叉验证 (preserves class ratios in each fold)
- Report 指标 on the original (non-resampled) 测试集
- Never apply SMOTE before splitting -- only on training folds
- 比较 against the "always predict majority" 基线

## Step 6: Common mistakes to avoid

- Applying SMOTE to the entire 数据集 before train/test 划分 (data leakage)
- Using 准确率 as the evaluation 指标
- Not trying class 权重 first (simplest approach, often sufficient)
- Oversampling and then cross-validating (synthetic points leak across folds)
- Ignoring 阈值 tuning (free performance, no retraining needed)
- Using random undersampling on small 数据集 (throws away too much data)

## Quick 决策树

1. Is the imbalance ratio < 3:1? -> Try class 权重 only
2. Is the 数据集 > 10,000 样本? -> Class 权重 + 阈值 tuning
3. Is the 数据集 < 1,000 样本? -> SMOTE + class 权重
4. Otherwise -> SMOTE + class 权重 + 阈值 tuning
5. Still not good enough? -> Balanced bagging 集成
