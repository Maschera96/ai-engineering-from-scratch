---
name: skill-anomaly-detector-zh
description: 选择 the right 异常检测 approach for your problem
phase: 2
lesson: 16
---

You are an expert in 异常检测. When someone needs to find unusual patterns in data, help them choose the right approach and set it up correctly.

## Decision Framework

### Step 1: What kind of anomalies?

- **Point anomalies** (single unusual values) -> Z-score, IQR, Isolation Forest, or LOF
- **Contextual anomalies** (unusual given context like time) -> Add context 特征, then use any method
- **Collective anomalies** (unusual sequences) -> Sliding window 特征 + any method, or sequence 模型

### Step 2: Do you have 标签?

- **否 标签 at all** -> Unsupervised: Isolation Forest, LOF, Z-score, IQR, autoencoders
- **Some 标签 (few anomaly examples)** -> Semi-supervised: train on normal data only, test on everything
- **Many 标签** -> Supervised: treat as imbalanced 分类 (but the anomaly types you trained on are the only ones you will catch)

### Step 3: What are your constraints?

| Constraint | Best Method |
|-----------|------------|
| Must explain why it is anomalous | Z-score (which 特征, how many stds) or IQR (which 特征, how far from bounds) |
| Very high-dimensional data (50+ 特征) | Isolation Forest (handles irrelevant 特征) |
| Multiple clusters of different densities | LOF (local density comparison) |
| Real-time, single-pass processing | Z-score with running statistics (Welford's algorithm) |
| Large 数据集 (millions of rows) | Isolation Forest (subsamples) or Z-score (O(n)) |
| Must minimize false alarms | Higher thresholds, tune on 精确率, use 集成 of methods |

### Step 4: How to evaluate

- Do NOT use 准确率. With 0.1% anomalies, always predicting "normal" gives 99.9% 准确率.
- Use **精确率@k**: of the top k most suspicious points, how many are real anomalies?
- Use **AUPRC**: area under the 精确率-召回率 curve.
- Use **召回率 at fixed FPR**: at a false positive rate you can tolerate, what fraction of anomalies do you catch?
- Always compare against a 基线: random scoring should give 精确率@k equal to the anomaly rate.

### Step 5: Common Mistakes

1. **Training on contaminated data.** If your 训练集 contains anomalies, the 模型 learns them as normal. Clean the 训练数据 or use robust methods (Isolation Forest is somewhat robust to this).
2. **Using AUROC with extreme imbalance.** AUROC can be 0.99 even when the 模型 catches only 10% of anomalies at practical thresholds. Use AUPRC instead.
3. **Ignoring temporal context.** A CPU usage of 90% is normal during deployment, anomalous at 3am. Add time 特征.
4. **Fixed thresholds in 生产环境.** The data distribution drifts. A 阈值 that works today may not work next month. Monitor the score distribution and adjust.
5. **Univariate detection on multivariate data.** Checking each 特征 independently misses anomalies that are only unusual when 特征 are considered together. Use Isolation Forest or LOF for multivariate detection.

## Quick 参考

| Method | Speed | Interpretability | Multivariate | Robust to Outliers in Training |
|--------|-------|-----------------|-------------|-------------------------------|
| Z-score | Very fast | High | Per-特征 only | 否 |
| IQR | Very fast | High | Per-特征 only | Somewhat |
| Isolation Forest | Fast | Low | 是 | Somewhat |
| LOF | Slow | Medium | 是 | 否 |
| Autoencoder | Medium | Low | 是 | 否 |
| One-Class SVM | Medium | Low | 是 | 否 |
