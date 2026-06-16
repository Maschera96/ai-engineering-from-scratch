---
name: skill-regression-zh
description: 选择 the right 回归 approach based on data characteristics and problem constraints
version: 1.0.0
phase: 2
lesson: 2
tags: [regression, linear-regression, polynomial-regression, ridge, regularization]
---

# 回归 Strategy Guide

回归 predicts continuous values. The right approach depends on the relationship between 特征 and 目标, the number of 特征, and the risk of 过拟合.

## 决策清单

1. Is the relationship between 特征 and 目标 approximately linear?
   - 是: start with ordinary 线性回归
   - 否: try polynomial 特征 or a nonlinear 模型

2. How many 特征 do you have relative to 样本?
   - Few 特征, many 样本: ordinary 线性回归 works fine
   - Many 特征, few 样本: use 正则化 (Ridge or Lasso)
   - More 特征 than 样本: Lasso (L1) to select 特征, or Ridge (L2) to shrink all 权重

3. Do you need interpretability?
   - 是: 线性回归 with few 特征, or Lasso for automatic 特征选择
   - 否: polynomial 特征, or move to 树-based 模型 or neural networks

4. Is your 数据集 small (under 10,000 rows)?
   - Use the normal equation (closed-form solution) for speed
   - 交叉验证 is essential for reliable evaluation

5. Is your 数据集 large (millions of rows)?
   - Use stochastic 梯度下降 (SGD) or mini-batch 梯度下降
   - The normal equation is too slow due to O(n^3) matrix inversion

## When to use each approach

**Ordinary 线性回归**: 基线 for any 回归 task. Start here. If R-squared is acceptable and the 模型 is simple, stop here.

**Polynomial 回归**: the scatter plot shows a curve, not a line. Start with degree 2. Increase only if justified by validation performance. Degree > 5 almost always overfits.

**Ridge 回归 (L2)**: many correlated 特征. All 权重 shrink toward zero but none become exactly zero. Good when you believe all 特征 contribute.

**Lasso 回归 (L1)**: many 特征 and you suspect only a few matter. Lasso drives irrelevant 特征 权重 to exactly zero, performing automatic 特征选择.

**Elastic Net**: combines L1 and L2 penalties. Use when you have many correlated 特征 and want some 特征选择.

## 常见错误

- Skipping 特征 scaling before 梯度下降 (convergence becomes extremely slow)
- Using 测试集 performance to tune 超参数 (use 验证集 or 交叉验证)
- Fitting high-degree polynomials without checking validation 误差 (training R^2 always increases with degree)
- Ignoring 残差 plots (R^2 can be misleading if 残差 show patterns)
- Treating R^2 as the only 指标 (check 残差 distribution, MAE, and domain-specific thresholds)

## 速查表

| Method | When to use | 正则化 | 特征选择 |
|--------|------------|---------------|-------------------|
| OLS | 基线, few 特征 | None | Manual |
| Ridge | Many 特征, all relevant | L2 (shrink) | 否 |
| Lasso | Many 特征, few relevant | L1 (zero out) | Automatic |
| Elastic Net | Many correlated 特征 | L1 + L2 | Partial |
| Polynomial | Nonlinear relationship | Add Ridge/Lasso on top | Manual degree choice |
