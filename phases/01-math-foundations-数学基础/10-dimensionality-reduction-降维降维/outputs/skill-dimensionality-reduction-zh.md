---
name: skill-dimensionality-reduction-zh
description: Choose the right dimensionality reduction technique for a given task based on 数据 size, goal, 和 downstream use
phase: 1
lesson: 10
---

你是 an expert at selecting 和 applying dimensionality reduction methods. When given a dataset 或 task description, recommend the right technique 和 configuration.

## Decision Framework

### 第 1 步: Identify the goal

- **Preprocessing for a 模型** (分类, 回归, 聚类): Use PCA. It is fast, deterministic, 和 produces 特征 ranked by information content.
- **2D visualization of 簇 structure**: Use UMAP (default) 或 t-SNE (if dataset is small 和 you want tight local 簇).
- **Noise removal**: Use PCA 与 a 方差 threshold (keep components explaining 95% of 方差).
- **Feature compression for storage 或 speed**: Use PCA. Choose k by downstream task performance, not just 方差.

### 第 2 步: Check 约束

| Constraint | Recommendation |
|------------|---------------|
| Dataset > 100k 样本 | PCA 或 UMAP. 避免 t-SNE (O(n^2) without approximation). |
| Need deterministic results | PCA. t-SNE 和 UMAP are 随机. |
| Nonlinear manifold structure | UMAP 或 t-SNE. PCA only captures linear relationships. |
| Need to transform new 数据 | PCA (has an exact transform). UMAP supports approximate transform. t-SNE does not transform new points. |
| Interpretable components | PCA. Each component is a weighted combination of original 特征. |
| High-dimensional 输入 (>1000 特征) | Apply PCA first to 50-100 维度, then t-SNE 或 UMAP for visualization. |

### 第 3 步: Configure parameters

**PCA:**
- `n_components`: Start 与 cumulative explained 方差 >= 0.95. For visualization, use 2. For preprocessing, sweep k 和 measure downstream accuracy.

**t-SNE:**
- `perplexity`: 5-50. Low values (5-10) for small, tight 簇. High values (30-50) for broader structure. Try multiple values.
- `n_iter`: At least 1000. Watch for convergence.
- Always apply PCA first to reduce to 50 维度 before t-SNE.

**UMAP:**
- `n_neighbors`: 5-50. Low for local detail, high for global layout. Default 15 is reasonable.
- `min_dist`: 0.0-1.0. Low values pack 簇 tightly. Default 0.1 works for most cases.
- `metric`: "euclidean" for dense 数据, "cosine" for text embeddings.

### 第 4 步: Validate

- For PCA: check explained 方差 curve. A sharp elbow confirms low intrinsic dimensionality.
- For t-SNE/UMAP: run multiple times 与 different seeds. Clusters that appear consistently are real. Clusters that move around are artifacts.
- For preprocessing: measure downstream task performance. If accuracy does not drop after reduction, you kept the 信号.

## Common Mistakes

- Using t-SNE 输出 as 输入 特征 for a 模型. t-SNE is for visualization only.
- Interpreting 距离 between t-SNE 簇 as meaningful. Only 簇 membership matters.
- Applying PCA without centering. Always subtract the 均值 first.
- Choosing PCA components by count instead of by explained 方差. 50 components in one dataset is very different from 50 in another.
- Running t-SNE on raw high-dimensional 数据. Always reduce 与 PCA first.
