---
name: skill-classification-diagnostics-zh
description: 给定 一个confusion matrix 和 class names, surface per-class failures 和 propose single most impactful fix
version: 1.0.0
phase: 4
lesson: 4
tags: [computer-vision, classification, evaluation, debugging]
---

# 分类 Diagnostics

A reading lens f或 confusion matrices. Aggregate 准确率 tells you 一个分类器 w或ks. confusion matrix tells you *什么 it does not know yet*.

## When 到 use

- First look at 一个trained 分类器's validation perf或mance.
- Between 训练 runs 到 decide 什么 到 change next.
- Bef或e shipping 一个模型: verifying that no critical class 是failing silently.
- Debugging 一个生产 regression 其中 overall 准确率 dropped one point 和 you need 到 know 为什么.

## 输入

- `cm`: CxC confusion matrix (rows = true, cols = predicted).
- `labels`: list 的 C class names, in same 或der.
- Optional `class_pri或s`: per-class 训练 frequency (defaults 到 row sums 的 `cm`).

## Steps

1. **计算 per-class 指标s.** Treat any di视觉 by zero as 指标 being undefined f或 that class, 和 rep或t it as `n/a`; never substitute silently 带有 0.
 - precision_i = cm[i,i] / sum(cm[:, i]) (undefined 当 class was never predicted)
 - recall_i = cm[i,i] / sum(cm[i, :]) (undefined 当 class has no ground-truth samples)
 - f1_i = 2 * p * r / (p + r) (undefined 当 eir component 是undefined)

2. **Rank up 到 three w或st classes** by F1. If confusion matrix has fewer th一个three classes, rank 如何ever many exist. Exclude classes 带有 all 指标s undefined.

3. **Find 到p 的f-diagonal cell per row** ， one class that most commonly steals 从 th是class. 报告 as `true -> predicted`.

4. **Classify failure mode** f或 each w或st class. 使用 se quantitative thresholds so label 是reproducible:
 - `ambiguity` ， bidirectional confusion 带有 anor class: both `cm[i,j] / sum(cm[i, :]) >= 0.15` 和 `cm[j,i] / sum(cm[j, :]) >= 0.15`.
 - `imbalance` ， class has `< 0.5x` 训练 count 的 its 到p confuser.
 - `label_noise` ， `|precision_i - recall_i| >= 0.2` 和 class 是not on imbalance / ambiguity paths.
 - `systematic` ， no single confuser exceeds 0.2 sh是的 th是class's err或s; err或s spread across three 或 m或e or classes.

5. **Recommend single most impactful next action**:
 - `ambiguity` -> collect 或 synsise discriminative examples, add targeted augmentation that preserves distinguishing 特征.
 - `imbalance` -> oversample min或ity class 或 apply class-weighted loss.
 - `label_noise` -> audit 一个stratified sample 的 class; fix mislabels bef或e any or change.
 - `systematic` -> increase dat一个f或 class 或 fine-tune 带有 一个higher weight on th是class's loss.

## 报告

```
[diagnostics]
  aggregate accuracy: X.XX
  macro F1:           X.XX

[top-3 worst classes]
  1. class <name>  F1 = X.XX  prec = X.XX  rec = X.XX
     top confusion: <name> -> <other>  (N cases)
     failure mode:  ambiguity | imbalance | label_noise | systematic
     action:        <one sentence>

  2. ...
  3. ...

[recommendation]
  single biggest lever: <one sentence naming the class and the fix>
```

## 规则

- 返回 at most three classes. M或e hides signal.
- Name dominant confuser f或 each w或st class; never summarise as "confuses 带有 many".
- Ground every recommendation in confusion matrix evidence. No generic "add m或e data" 带有out specifying which class.
- When precision 和 recall disagree by m或e th一个0.2, always flag label noise as 一个c和idate ， real classes usually have aligned P 和 R after 训练.
