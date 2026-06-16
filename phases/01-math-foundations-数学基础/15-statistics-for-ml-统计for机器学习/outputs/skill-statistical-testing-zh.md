---
name: skill-statistical-testing-zh
description: Choose the right statistical test for comparing ML 模型 和 evaluating experiments
version: 1.0.0
phase: 1
lesson: 15
tags: [statistics, hypothesis-testing, model-comparison]
---

# Statistical Testing for ML

How to pick the right test when comparing 模型, running A/B experiments, 或 validating results.

## Decision Checklist

1. What are you comparing? Means, proportions, 分布, 或 correlations?
2. How many groups? One 样本 vs reference, two groups, 或 multiple groups?
3. Are observations paired (same test set, same folds) 或 independent?
4. Is the 数据 normally distributed? If n < 30 和 not clearly normal, use non-parametric.
5. Is the 数据 continuous, ordinal, 或 categorical?
6. How many tests are you running? Apply correction if more than one.

## Decision tree

```text
Comparing means?
  Two groups?
    Paired (same data splits)? --> Paired t-test (or Wilcoxon signed-rank if non-normal)
    Independent? --> Welch's t-test (or Mann-Whitney U if non-normal)
  Multiple groups?
    Paired? --> Repeated measures ANOVA (or Friedman test)
    Independent? --> One-way ANOVA (or Kruskal-Wallis)

Comparing proportions?
  Two groups? --> Chi-squared test or Fisher's exact test (small n)
  Multiple groups? --> Chi-squared test

Comparing distributions?
  Is one distribution a reference? --> Kolmogorov-Smirnov test
  Are both empirical? --> Two-sample KS test

Measuring association?
  Both continuous, roughly normal? --> Pearson correlation
  Ordinal or non-normal? --> Spearman rank correlation
  Categorical x Categorical? --> Chi-squared test of independence

Running many tests?
  Apply Bonferroni correction: alpha_adjusted = alpha / number_of_tests
  Or use Holm-Bonferroni (less conservative, still controls family-wise error)
```

## When to use each test

| Test | Data type | Assumptions | ML use case |
|---|---|---|---|
| Paired t-test | Continuous, paired | Normal differences | Compare 2 模型 on same k-fold splits |
| Wilcoxon signed-rank | Continuous/ordinal, paired | 无 (non-parametric) | Compare 2 模型, small k (5-10 folds) |
| Welch's t-test | Continuous, independent | Roughly normal | Compare 模型 on two separate datasets |
| Mann-Whitney U | Continuous/ordinal, independent | 无 | Compare latency 分布 |
| ANOVA | Continuous, 3+ groups | Normal, equal 方差 | Compare multiple 模型 architectures |
| Kruskal-Wallis | Continuous/ordinal, 3+ groups | 无 | Compare multiple 模型, non-normal metrics |
| Chi-squared | Categorical counts | Expected count >= 5 | Compare class 分布, confusion 矩阵 |
| Fisher's exact | Categorical counts | Small 样本 | Rare event comparison |
| KS test | Continuous | 无 | Check if predictions follow expected 分布 |
| Bootstrap CI | Any statistic | 无 | Confidence interval for AUC, F1, any metric |
| McNemar's test | Paired binary | 无 | Compare two classifiers on same test set |

## Model comparison recipe

1. Define metric 和 significance level (alpha = 0.05) before running experiments.
2. Run both 模型 on the same k-fold cross-validation splits (k = 5 或 10).
3. Collect paired scores: (a_1, b_1), (a_2, b_2), ..., (a_k, b_k).
4. Compute differences: d_i = b_i - a_i.
5. Run paired test (Wilcoxon for k <= 10, paired t-test for k > 10 或 normal diffs).
6. Report: p-value, 均值 difference, 95% confidence interval, effect size (Cohen's d).
7. If p < alpha AND effect size is meaningful, the difference is real 和 worth acting on.

## Common mistakes

- Using an independent test when 数据 is paired. If both 模型 were evaluated on the same test folds, you must use a paired test. Independent tests throw away the pairing 和 lose statistical power.
- Reporting p < 0.05 without effect size. A statistically significant 0.1% accuracy improvement is not worth deploying. Always compute Cohen's d 或 the raw 均值 difference.
- Comparing 模型 across different test sets. The test set MUST be identical for both 模型. Different test sets make comparison meaningless.
- Running 20 comparisons 和 reporting the best one without Bonferroni correction. With 20 tests at alpha = 0.05, you expect 1 假 正 by chance.
- Using accuracy on imbalanced 数据. On a 99% majority class, a trivial classifier achieves 99%. Use F1, precision-recall AUC, 或 Matthews 相关性 coefficient.
- Treating cross-validation folds as independent 样本. They share training 数据, which violates the independence assumption. The corrected resampled t-test accounts for this.

## Quick reference: effect size interpretation

| Cohen's d | Interpretation |
|---|---|
| 0.2 | Small effect |
| 0.5 | Medium effect |
| 0.8 | Large effect |
| > 1.0 | Very large effect |

| What to report | Why |
|---|---|
| p-value | Is the difference real? |
| Confidence interval | How big could the difference be? |
| Effect size (Cohen's d) | Is the difference meaningful? |
| 样本 size (n 或 k folds) | Can we trust the result? |
