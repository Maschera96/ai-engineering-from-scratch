---
name: skill-feature-selector-zh
description: Quick reference 决策树 for choosing the right 特征选择 method
version: 1.0.0
phase: 2
lesson: 18
tags: [feature-selection, mutual-information, rfe, lasso, tree-importance]
---

# 特征选择 Strategy

A quick reference for picking and applying the right 特征选择 method.

## Step 1: Start with cleanup

Before applying any method, remove obviously useless 特征:

- **Constant 特征**: 方差 = 0. Remove them.
- **Near-constant 特征**: 方差 < 0.01 (or your 阈值). Remove them.
- **Duplicate 特征**: identical columns. Keep one, drop the rest.
- **ID columns**: unique per row, carry no generalizable information. Remove them.

This takes seconds and can eliminate 10-30% of 特征 in messy real-world 数据集.

## Step 2: 选择 a method based on your situation

### Quick 决策树

1. **< 50 特征?** Start with mutual information ranking. Keep top K.
2. **50 - 500 特征?** Use 方差 阈值 first, then L1 (Lasso) if using a linear 模型, or 树 importance if using 树.
3. **> 500 特征?** Chain methods: 方差 阈值 -> mutual information filter (top 50%) -> RFE on survivors.
4. **Need interpretability?** L1 正则化 gives you exact zero/nonzero. 树 importance gives ranked scores.
5. **Need to capture nonlinear relationships?** Mutual information or 树-based importance. Avoid L1 (linear only).
6. **Need 特征 interactions?** RFE or 树-based importance. Filter methods miss interactions.

### Method 参考

| Method | When to Use | When to Avoid |
|--------|------------|---------------|
| 方差 阈值 | Always, as a first step | Never skip this |
| Mutual information | Quick ranking, nonlinear relationships | When you need 特征 interaction detection |
| RFE | Thorough selection, moderate 特征 count | Very expensive 模型, > 1000 特征 |
| L1 / Lasso | Linear 模型, fast embedded selection | Nonlinear problems, highly correlated 特征 |
| 树 importance | Nonlinear relationships, 特征 interactions | Biased by high-cardinality 特征 |
| Permutation importance | 模型-agnostic validation, final check | Too slow for initial screening |

## Step 3: Validate your selection

- 比较 模型 performance with selected 特征 vs all 特征
- Use 交叉验证, not a single train/test 划分
- If performance drops by more than 1-2%, you may have removed useful 特征
- If performance improves, you successfully removed noise

## Step 4: Handle common pitfalls

### Correlated 特征
- L1 arbitrarily picks one from a correlated group and zeros the others
- 计算 the correlation matrix first and decide which correlated 特征 to keep
- 树 importance spreads importance across correlated 特征

### Data leakage
- Fit 特征选择 on 训练数据 only
- Apply the same selection to 测试数据
- In 交叉验证, 特征选择 must happen inside each fold

### 过拟合 to 特征选择
- RFE with too many iterations can overfit to the 训练集
- Validate on held-out data, not the data used for selection
- Use stability selection (repeat on subsamples) for more robust results

## Step 5: 生产环境 checklist

- [ ] 方差 阈值 applied as first filter
- [ ] 特征选择 fitted on 训练数据 only
- [ ] Selected 特征 documented (names, method used, scores)
- [ ] Performance compared: selected 特征 vs all 特征
- [ ] Cross-validated, not single-划分 evaluation
- [ ] 特征选择 integrated into the training 流水线 (not done manually)
- [ ] Monitoring in place for 特征 drift (selected 特征 may become stale)
