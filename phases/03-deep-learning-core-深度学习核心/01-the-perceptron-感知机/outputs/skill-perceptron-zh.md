---
name: skill-perceptron-zh
description: Understand perceptron pattern 和 当 到 使用 single-层 vs multi-层 architectures
version: 1.0.0
phase: 3
lesson: 1
tags: [perceptron, neural-networks, classification, deep-learning]
---

# 感知机 Pattern

A perceptron computes a weighted sum of 输入 plus a 偏置, 然后 applies a 步骤 函数 到 produce a binary 输出. It 是 fundamental unit of 神经网络.

```
output = step(w1*x1 + w2*x2 + ... + wn*xn + bias)
```

## When a single perceptron 是 enough

- 问题 是 linearly separable: a straight line (或 hyperplane) can divide two classes
- Logic gates: AND, OR, NOT, NAND
- Simple threshold decisions: "是 score above X?"
- Binary classifiers 在 数据 that clusters into two non-overlapping regions

## When 你 need multiple 层

- 问题 是 不 linearly separable: 没有 single line can separate classes
- XOR 和 parity problems
- Any 任务 requiring "这 but 不 that" reasoning (combinations of conditions)
- Real-world 分类: images, text, audio - almost always non-线性

## Decision 检查清单

1. Plot 或 inspect 你的 数据. Can 你 draw a single straight 边界 between classes?
   - Yes: single perceptron works
   - No: 你 need at least two 层
2. Can 问题 be decomposed into AND/OR of simpler 线性 decisions?
   - 这 decomposition tells 你 minimum network structure
   - XOR = (A OR B) AND (NOT (A AND B)) = 3 perceptrons 在 2 层
3. For problems 用 more than two classes, 你 need one 输出 node per class

## 训练 规则

```
error = expected - predicted
weight_new = weight_old + learning_rate * error * input
bias_new = bias_old + learning_rate * error
```

If 预测 是 correct, nothing changes. If wrong, 权重 shift 到 降低 错误. 这 only works 用于 single-层 perceptrons. Multi-层 networks require 反向传播.

## Common mistakes

- Trying 到 learn non-线性 patterns 用 a single perceptron (it will never converge)
- Setting 学习率 too high (权重 oscillate) 或 too low (训练 takes forever)
- Forgetting 偏置 term (不用 it, 决策 边界 must pass through origin)
- Confusing perceptron convergence (guaranteed 用于 linearly separable 数据) 用 general 神经网络 convergence (不 guaranteed)
