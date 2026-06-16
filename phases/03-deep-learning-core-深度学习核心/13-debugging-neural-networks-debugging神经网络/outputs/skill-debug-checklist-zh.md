---
name: skill-debug-checklist-zh
description: 用于调试神经网络训练失败的决策树检查清单
version: 1.0.0
phase: 3
lesson: 13
tags: [debugging, neural-networks, training, diagnostics, deep-learning]
---

# 神经网络 调试 Checklist

当训练出问题时使用的系统化调试流程。按顺序完成这些检查 -- 大多数缺陷会在前 3 步被发现.

## Before 训练 (prevent 缺陷)

1. 打印 模型 架构 和 parameter count. Does size make sense 用于 你的 数据?
2. 运行 a single 前向传播 用 random 输入. Does 输出 形状 match 你的 target 形状?
3. 检查 that 标签 是 correct dtype (CrossEntropy损失 needs Long, BCE损失 needs Float)
4. 确认 数据 归一化: 输入 should have 均值 near 0 和 std near 1
5. 打印 5 random (输入, 标签) pairs. Do 标签 match what 你 expect?
6. Confirm 训练/test split has 没有 duplicate 样本

## Overfit-one-批次 test (60 seconds, catches 80% of 缺陷)

1. Take 8-32 样本 从 你的 训练 set
2. 训练 用于 200 步骤 用 a reasonable 学习率
3. 损失 should approach 0. 训练 准确率 should hit 100%
4. If it fails: 缺陷 是 在 你的 模型, 损失 函数, 或 训练循环 -- 不 你的 数据 或 hyper参数
5. If it passes: proceed 到 full 训练

## 损失 不 decreasing

1. 检查 学习率. Try 3 值: current/10, current, current*10
2. 打印 梯度 norms per 层. All zeros means dead network 或 detached graph
3. 检查`requires_grad=True`在 参数. 检查 that`loss.backward()`是 called
4. 检查 that`optimizer.zero_grad()`是 called 之前`loss.backward()`
5. 检查 that`optimizer.step()`是 called 之后`loss.backward()`
6. 确认 模型 参数 是 passed 到 优化器:`optimizer = Adam(model.parameters())`

## 损失 是 NaN 或 Inf

1. 降低 学习率 by 10x
2. 加入 epsilon 到 all log() calls:`torch.log(x + 1e-7)`
3. 加入 epsilon 到 all division:`x / (y + 1e-8)`
4. Clamp 预测s:`torch.clamp(pred, 1e-7, 1 - 1e-7)`之前 BCE 损失
5. 使用`torch.autograd.detect_anomaly()`到 find exact operation
6. 检查 用于 NaN 在 输入 数据:`assert not torch.isnan(x).any()`

## 损失 oscillating

1. 降低 学习率 by 3-10x
2. 增加 批次 size (reduces 梯度 噪声)
3. 加入 梯度 clipping:`torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)`
4. 切换 从 SGD 到 Adam (adaptive LR per parameter)
5. 加入 学习率 warmup 用于 first 5-10% of 训练

## 过拟合 (训练 acc high, test acc low)

1. 加入 dropout (开始 用 p=0.1, 增加 到 0.5)
2. 加入 权重衰减 到 优化器:`Adam(params, weight_decay=1e-4)`
3. 降低 模型 size (fewer 层 或 narrower 层)
4. 加入 数据 增强
5. 使用 early stopping: 停止 当 验证 损失 increases 用于 5+ 轮次
6. 检查 用于 数据 leakage between 训练 和 test sets

## 欠拟合 (both 训练 和 test acc low)

1. 增加 模型 容量 (more 层, wider 层)
2. 训练 用于 more 轮次
3. 增加 学习率 (carefully)
4. 移除 正则化 temporarily 到 确认 模型 can learn
5. 检查 that 你的 模型 是 expressive enough 用于 任务

## Dead ReLU neurons

1. 检查 fraction of zero 激活s per 层. >50% 是 a 问题
2. 切换 到 LeakyReLU(0.01) 或 GELU
3. 使用 Kaiming initialization 用于 权重
4. 降低 学习率 (large updates can push neurons into dead zone)
5. 加入 批归一化 之前 激活 函数

## 快速参考: 学习率 starting points

|优化器|Task|Starting LR|
|-----------|------|------------|
|Adam|训练 从零实现|1e-3|
|Adam|Fine-tuning pretrained|1e-5|
|SGD + momentum|训练 从零实现|1e-1|
|SGD + momentum|Fine-tuning pretrained|1e-3|
|AdamW|Transformer 训练|3e-4|

## 快速参考: 批次 size effects

|Batch size|梯度 噪声|Memory|Generalization|
|-----------|---------------|--------|---------------|
|8-16|High (noisy)|Low|Often better|
|32-64|Moderate|Moderate|Good 默认|
|128-256|Low (smooth)|High|May need warmup|
|512+|Very low|Very high|Needs LR scaling|

## When nothing works

1. Simplify 模型 到 1 hidden 层. Does it learn?
2. Simplify 数据 到 100 样本. Does it overfit?
3. Replace 你的 损失 用 MSE. Does it converge?
4. Replace 你的 优化器 用 SGD(lr=0.01). Does it make progress?
5. Replace 你的 数据 用 synthetic 数据 (e.g., y = x[0] > 0). Does it learn?
6. If none of these work: 缺陷 是 在 code 你 是 不 looking at (数据 loading, preprocessing, 张量 shapes)
