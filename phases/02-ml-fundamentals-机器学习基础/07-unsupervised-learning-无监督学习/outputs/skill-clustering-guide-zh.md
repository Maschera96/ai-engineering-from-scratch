---
name: skill-clustering-guide-zh
description: 选择 the right 聚类 algorithm based on data shape, noise, and constraints
version: 1.0.0
phase: 2
lesson: 7
tags: [clustering, k-means, dbscan, hierarchical, gmm, unsupervised]
---

# 聚类 Algorithm Selection Guide

聚类 has no single best algorithm. The right choice depends on cluster shape, whether you know the number of clusters, how much noise is in the data, and how large the 数据集 is.

## 决策清单

1. Do you know the number of clusters?
   - 是: K-Means or GMM
   - 否: DBSCAN (finds clusters automatically), or hierarchical (cut the dendrogram at different levels)

2. What shape are the clusters?
   - Roughly spherical (blob-like): K-Means
   - Elliptical with different sizes: GMM
   - Arbitrary shapes (crescents, rings, chains): DBSCAN
   - Nested or hierarchical: hierarchical 聚类

3. Does the data contain noise or outliers?
   - 是: DBSCAN (标签 noise points explicitly) or GMM (low-概率 points are outliers)
   - 否: K-Means is fine

4. Do you need soft assignments (概率)?
   - 是: GMM gives P(cluster | data point) for each cluster
   - 否: K-Means or DBSCAN give hard assignments

5. How large is the 数据集?
   - Under 10,000: any algorithm works
   - 10,000 to 1,000,000: K-Means (fast), Mini-Batch K-Means (faster)
   - Over 1,000,000: Mini-Batch K-Means or BIRCH. Hierarchical is too slow.

## When to use each approach

**K-Means**: the default starting point. Fast (O(n * k * iterations)), simple, and good enough for many problems. Use the elbow method or silhouette score to pick K. Limitations: assumes spherical clusters, sensitive to initialization (use K-Means++ or run multiple times), cannot handle varying cluster sizes well.

**DBSCAN**: best for discovering clusters of arbitrary shape and automatically detecting outliers. Two 参数: eps (neighborhood radius) and min_samples (minimum density). Does not require specifying K. Limitations: struggles when clusters have very different densities, and tuning eps can be tricky. Use a k-距离 plot to estimate eps: compute the 距离 to each point's k-th nearest neighbor, sort, and look for an elbow.

**Hierarchical (Agglomerative)**: builds a 树 of merges. Useful when you want to explore cluster structure at multiple granularities (cut the dendrogram at different heights). Ward's linkage works best for compact clusters. Single linkage finds elongated clusters but is sensitive to noise. Limitations: O(n^2) memory and O(n^3) time, so impractical for large 数据集.

**GMM (Gaussian Mixture 模型)**: soft 聚类 with probabilistic assignments. 模型 each cluster as a Gaussian distribution with its own mean and covariance. Better than K-Means when clusters are elliptical or overlapping. Use BIC (Bayesian Information Criterion) to select the number of components. Limitations: assumes Gaussian distributions, can fail on non-convex shapes, sensitive to initialization.

## Evaluating cluster quality (no 标签)

| 指标 | What it measures | Range | Use when |
|--------|-----------------|-------|----------|
| Silhouette score | Cohesion vs separation | -1 to 1 (higher is better) | Comparing K values or algorithms |
| Inertia (within-cluster SS) | Tightness of clusters | 0 to inf (lower is better) | Elbow method for K-Means |
| BIC / AIC | 模型 fit with complexity penalty | Lower is better | Choosing number of GMM components |
| Calinski-Harabasz index | Ratio of between to within 方差 | Higher is better | Quick comparison |
| Davies-Bouldin index | Average similarity between clusters | Lower is better | Penalizes overlapping clusters |

## 常见错误

- Running K-Means without scaling 特征 (特征 on larger scales dominate the 距离 calculation)
- Picking K by eyeballing data in 2D when the actual data is high-dimensional (use silhouette scores)
- Using K-Means on non-spherical clusters (crescent or ring-shaped data needs DBSCAN)
- Setting DBSCAN eps too large (everything in one cluster) or too small (everything is noise)
- Treating cluster 标签 as ground truth (聚类 is exploratory; validate with domain knowledge)
- Running hierarchical 聚类 on 数据集 with more than 20,000 points (memory and time explode)

## 速查表

| Algorithm | Cluster shape | Finds K | Handles noise | Soft assignments | Scalability |
|-----------|--------------|---------|---------------|-----------------|-------------|
| K-Means | Spherical | 否 (you set K) | 否 | 否 | Millions |
| Mini-Batch K-Means | Spherical | 否 | 否 | 否 | Tens of millions |
| DBSCAN | Arbitrary | 是 | 是 | 否 | Hundreds of thousands |
| Hierarchical | Any (linkage-dependent) | Flexible (cut dendrogram) | Depends on linkage | 否 | Under 20k |
| GMM | Elliptical | 否 (you set K) | Partial (low 概率) | 是 | Under 100k |
| HDBSCAN | Arbitrary | 是 | 是 | Partial | Hundreds of thousands |
