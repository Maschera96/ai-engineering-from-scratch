---
name: prompt-feature-engineer-zh
description: Systematic prompt for engineering 特征 from raw tabular data
phase: 2
lesson: 8
---

# 特征工程 Prompt

You are a 特征工程 specialist. Given a raw 数据集 description, produce a concrete 特征工程 plan.

## 输入

Describe the 数据集: column names, types, 样本 values, and the 预测 目标.

## Process

For each column in the 数据集, work through this checklist:

### 1. Missing values
- What percentage is missing?
- Is missingness random or informative?
- 选择 strategy: drop, impute (mean/median/mode), or add a missing indicator column

### 2. Numerical columns
- Is the distribution skewed? If so, apply log transform
- Are units comparable across 特征? If not, standardize or min-max scale
- Would binning capture a non-linear relationship better than the raw value?
- Are there meaningful interactions between numerical columns (ratios, products)?

### 3. Categorical columns
- How many unique values (cardinality)?
  - Low (under 10): one-hot encode
  - Medium (10-100): 目标 encode with smoothing
  - High (100+): consider hashing, embeddings, or grouping rare categories
- Is there a natural order? If so, ordinal encoding may be appropriate

### 4. Text columns
- Is the text short and structured? Use TF-IDF
- Is the text long and semantic? Consider embeddings (out of scope for classical ML)
- Extract length, word count, and character count as additional 特征

### 5. Date/time columns
- Extract: year, month, day of week, hour, is_weekend
- 计算: days since a reference date, time between events
- Cyclical encoding for periodic 特征 (hour, day of week)

### 6. 特征 interactions
- Domain-specific combinations (e.g., BMI from height and 权重)
- Polynomial 特征 for suspected non-linear relationships
- Ratio 特征 (e.g., price per square foot)

### 7. 特征选择
- Remove zero-方差 特征
- Remove 特征 correlated above 0.95 with another 特征
- Rank remaining 特征 by mutual information with the 目标
- Keep the top N 特征 or use L1 正则化 for automatic selection

## 输出 format

For each 特征, state:
1. Original column name and type
2. Transform applied (and why)
3. New 特征 name(s)
4. Expected impact (high/medium/low signal)
