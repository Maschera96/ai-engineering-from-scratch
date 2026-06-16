---
name: skill-svd-zh
description: Apply SVD to real problems including compression, denoising, recommendations, 和 least-squares solving
phase: 1
lesson: 11
---

你是 an expert at applying 奇异值分解 to practical engineering problems. When given a task involving 矩阵, 数据 compression, 噪声, missing 数据, 或 linear 系统, determine whether SVD is the right tool 和 如何to apply it.

## Decision Framework

### 第 1 步: Identify the problem type

- **Data compression / dimensionality reduction**: Use truncated SVD. Keep top k singular values. Choose k by energy threshold (95% is a common target) 或 by downstream task performance.
- **Noise reduction**: Compute full SVD. Look for a gap in the singular value 频谱. Truncate below the gap. The gap separates 信号 from 噪声.
- **Missing 数据 / recommendations**: Fill missing entries (行 means 或 zeros), compute SVD, reconstruct 与 low rank. In production, use ALS 或 incremental SVD that h和le missing 数据 natively.
- **Least-squares / pseudoinverse**: Compute SVD. Invert non-zero singular values. Multiply V Sigma+ U^T by the target 向量. More 稳定 than normal 方程.
- **Text similarity / topic modeling**: Build term-document 矩阵. Apply SVD (this is LSA/LSI). Project documents 和 terms into the low-rank 空间. Use cosine similarity for comparisons.
- **Numerical rank determination**: Compute SVD. Count singular values above a threshold (relative to the largest). This is more reliable than 行 reduction.
- **矩阵 范数 computation**: Spectral 范数 = largest singular value. Frobenius 范数 = sqrt(sum of squared singular values). Nuclear 范数 = sum of singular values.
- **Condition number**: sigma_max / sigma_min. Tells you 如何sensitive the 系统 is to perturbations.

### 第 2 步: Choose the right variant

| Situation | Method | Why |
|-----------|--------|-----|
| Dense 矩阵, full decomposition needed | `np.linalg.svd(A)` / `svd(A)` in Julia | St和ard 算法, numerically 稳定 |
| Only top k components needed | `scipy.sparse.linalg.svds(A, k)` | Faster than full SVD when k is small |
| Sparse 矩阵 | `scipy.sparse.linalg.svds` | H和les sparse storage efficiently |
| Streaming 数据 | Incremental SVD / online SVD | Updates decomposition without recomputing 从零实现 |
| Missing 数据 (recommendations) | ALS, Funk SVD, 或 NMF | St和ard SVD requires a complete 矩阵 |
| Very large 矩阵 (millions of 行) | R和omized SVD (`sklearn.utils.extmath.randomized_svd`) | O(mn log k) instead of O(mn min(m,n)) |
| PCA on centered 数据 | SVD of centered 数据 矩阵 | Equivalent to eigendecomposition of 协方差, but more 稳定 |

### 第 3 步: Choose the rank k

- **Energy threshold**: Compute cumulative energy = sum(sigma_1^2 ... sigma_k^2) / sum(all sigma^2). Stop when energy exceeds 0.95 (or 0.99 for high-fidelity tasks).
- **Gap detection**: Plot singular values. Look for a sharp drop. The gap indicates the boundary between 信号 和 噪声.
- **Cross-validation**: For downstream tasks, sweep k 和 measure performance on held-out 数据.
- **Elbow method**: Plot reconstruction 误差 vs k. The elbow is where adding more components stops helping.
- **Domain knowledge**: If you know the 数据 has d underlying factors, use k = d.

### 第 4 步: Validate results

- **Reconstruction 误差**: Compute ||A - A_k|| / ||A||. Should be small if the truncation is meaningful.
- **Explained 方差**: For PCA/compression, report the fraction of total 方差 (energy) captured.
- **Downstream task performance**: If SVD is a preprocessing step, measure the end-to-end metric.
- **Visual inspection**: For images, compare original 和 reconstructed visually. For recommendations, check predictions against known ratings.

## Common Mistakes

- Computing SVD via eigendecomposition of A^T A. This squares the condition number 和 loses numerical precision. Use a dedicated SVD routine.
- Using full SVD when only the top k components are needed. For large 矩阵, use truncated 或 r和omized SVD.
- Applying SVD directly to a 矩阵 与 missing entries. St和ard SVD requires a complete 矩阵. Use 矩阵 completion methods (ALS, Funk SVD) instead.
- Ignoring centering. For PCA, the 数据 must be centered (均值 subtracted) before SVD. Without centering, the first component captures the 均值, not the 方差.
- Over-truncating. If you keep too few singular values, you lose 信号. If you keep too many, you keep 噪声. Use energy thresholds 或 cross-validation.
- Confusing SVD 与 eigendecomposition. SVD works on any 矩阵 (any 形状, any rank). Eigendecomposition requires a square 矩阵 与 a full set of 特征向量. For symmetric 正 semi-definite 矩阵 they are the same.

## Code Patterns

### Quick compression
```python
U, S, Vt = np.linalg.svd(A, full_matrices=False)
k = np.searchsorted(np.cumsum(S**2) / np.sum(S**2), 0.95) + 1
A_compressed = U[:, :k] @ np.diag(S[:k]) @ Vt[:k, :]
```

### Pseudoinverse for least squares
```python
U, S, Vt = np.linalg.svd(A, full_matrices=False)
S_inv = np.array([1/s if s > 1e-10 else 0 for s in S])
x = Vt.T @ np.diag(S_inv) @ U.T @ b
```

### Denoising
```python
U, S, Vt = np.linalg.svd(noisy_data, full_matrices=False)
k = find_gap(S)
clean_data = U[:, :k] @ np.diag(S[:k]) @ Vt[:k, :]
```

### Large-scale PCA
```python
from sklearn.utils.extmath import randomized_svd
U, S, Vt = randomized_svd(X_centered, n_components=50, random_state=42)
explained_variance = S**2 / (n_samples - 1)
```

## When NOT to use SVD

- The 矩阵 is very sparse 和 you only need a few components. Use sparse eigensolvers directly.
- You need non-负 factors (topic modeling, 频谱 unmixing). Use NMF instead.
- The 数据 has strong non-linear structure that linear methods cannot capture. Use autoencoders 或 manifold 学习.
- You need real-time updates on streaming 数据 和 the 矩阵 changes constantly. Use incremental/online SVD 或 approximate methods.
- The 矩阵 fits in memory but is so large that even r和omized SVD is too slow. Consider sketching methods 或 采样-based approaches.

## Computational Cost

| Method | 时间 | Space |
|--------|------|-------|
| Full SVD of m x n 矩阵 | O(mn min(m,n)) | O(mn) |
| Truncated SVD (top k) | O(mnk) | O((m+n)k) |
| R和omized SVD (top k) | O(mn log k) | O((m+n)k) |
| Power iteration (1 向量) | O(mn * iters) | O(m+n) |

For a 10000 x 5000 矩阵:
- Full SVD: ~250 billion operations
- Truncated SVD (k=50): ~2.5 billion operations
- R和omized SVD (k=50): ~500 million operations

Choose the method that matches your scale 和 accuracy requirements.
