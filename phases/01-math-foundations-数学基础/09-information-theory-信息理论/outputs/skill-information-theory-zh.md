---
name: skill-information-theory-zh
description: Apply information theory concepts to ML 损失 函数, 模型 evaluation, 和 特征 selection
version: 1.0.0
phase: 1
lesson: 9
tags: [information-theory, entropy, loss-functions]
---

# Information Theory for ML

当to use 熵, cross-熵, KL 散度, 和 mutual information in machine 学习 系统.

## Decision Checklist

1. Measuring uncertainty in a single 分布? Use **熵**.
2. Measuring 如何well a 模型 approximates 真 labels? Use **cross-熵** (this is your 分类 损失).
3. Measuring 距离 between two 分布? Use **KL 散度**.
4. Checking if two variables are related? Use **mutual information**.
5. Reporting language 模型 quality? Use **perplexity** (exponential of cross-熵).
6. Distilling one 模型 into another? Minimize **KL 散度** from teacher to student.

## When to use each measure

| Measure | Formula | Use case | ML application |
|---|---|---|---|
| 熵 H(P) | -sum(p log p) | How uncertain is this 分布? | Data complexity, maximum 熵 模型 |
| Cross-熵 H(P,Q) | -sum(p log q) | How good is 模型 Q at predicting 真 P? | Classification 损失, language 模型 损失 |
| KL 散度 D(P\|\|Q) | sum(p log(p/q)) | How different are P 和 Q? | VAE 损失 (ELBO), knowledge distillation, RLHF |
| Mutual information I(X;Y) | H(X) - H(X\|Y) | How much does Y tell us 约 X? | Feature selection, representation 学习 |
| Perplexity | exp(H(P,Q)) 或 2^H | How confused is the 模型? | Language 模型 evaluation |
| Conditional 熵 H(X\|Y) | -sum(p(x,y) log p(x\|y)) | Remaining uncertainty in X after knowing Y | Feature informativeness |

## Key relationships

```
Cross-entropy  = Entropy + KL divergence
H(P, Q)        = H(P)   + D_KL(P || Q)

Since H(P) is constant during training:
  Minimizing cross-entropy = Minimizing KL divergence

Mutual information = Entropy - Conditional entropy
I(X; Y) = H(X) - H(X|Y) = H(Y) - H(Y|X)

Perplexity = exp(cross-entropy in nats)
           = 2^(cross-entropy in bits)
```

## Quick reference: formulas 和 units

| Formula | Bits (log base 2) | Nats (log base e) |
|---|---|---|
| Information: -log(p) | -log2(p) | -ln(p) |
| 熵: -sum(p log p) | bits | nats |
| 1 nat = | 1.4427 bits | 1 nat |
| PyTorch default | -- | nats |
| Information theory papers | bits | -- |

## Interpreting values

| 熵 value | What it means |
|---|---|
| 0 | Deterministic. One outcome has 概率 1. |
| log(n) | Maximum uncertainty. Uniform 分布 over n outcomes. |
| Low | 分布 is peaked. Model is confident. |
| High | 分布 is flat. Model is uncertain. |

| Perplexity value | Language 模型 quality |
|---|---|
| 1 | Perfect prediction (never happens in practice) |
| 10 | Choosing among ~10 equally likely tokens on average |
| 50 | GPT-2 level on st和ard benchmarks |
| < 10 | State-of-the-art for well-represented domains |

## Common mistakes

- Computing KL 散度 和 treating it as symmetric. D_KL(P||Q) != D_KL(Q||P). For a symmetric measure, use Jensen-Shannon 散度: JS = 0.5 * KL(P||M) + 0.5 * KL(Q||M) where M = 0.5*(P+Q).
- Forgetting that cross-熵 与 one-hot labels simplifies to -log(p_true_class). You do not need to sum over all classes when the 真 分布 is one-hot.
- Using log base 2 in code but reporting nats (or vice versa). PyTorch uses natural log by default. Multiply by log2(e) = 1.4427 to convert nats to bits.
- Computing 熵 of an empty 或 zero-概率 event. Convention: 0 * log(0) = 0, because lim(p->0) p*log(p) = 0.
- Comparing perplexity across different vocabularies. A 模型 与 vocab size 50k 和 perplexity 30 is not directly comparable to one 与 vocab size 10k 和 perplexity 30.

## Where each concept appears in production ML

| Concept | Where you see it |
|---|---|
| Cross-熵 损失 | Every 分类 模型 (nn.CrossEntropyLoss) |
| KL 散度 | VAE ELBO, PPO clipping, knowledge distillation |
| 熵 regularization | Exploration bonus in RL (higher 熵 = more exploration) |
| Mutual information | Feature selection, InfoNCE 损失 (contrastive 学习) |
| Perplexity | Language 模型 benchmarks (lower = better) |
| Label smoothing | Replaces one-hot 与 soft targets, reduces cross-熵 overconfidence |
| Temperature scaling | Divides logits by T before softmax, controls 熵 of 输出 |
