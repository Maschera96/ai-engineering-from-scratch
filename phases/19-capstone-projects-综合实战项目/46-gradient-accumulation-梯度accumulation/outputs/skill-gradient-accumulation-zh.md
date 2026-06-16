---
name: gradient-accumulation-zh
description: 通过缩放 micro-batch loss，并在每个窗口只执行一次 optimizer step，在设备内存无法容纳的情况下训练更大的 effective batch。
version: 1.0.0
phase: 19
lesson: 46
tags: [training, batch-size, distributed, scaling]
---

## 何时使用

Effective batch 是平滑梯度并匹配 learning rate schedule 的关键杠杆。当你无法在单次 forward pass 中容纳它时，就用这套 recipe。

## Recipe

1. 将 `micro_batch` 设为内存能容纳且能让 accelerator 饱和的最大值。
2. 从 learning rate schedule 选择 `effective_batch`。
3. 设置 `accum_steps = effective_batch // (micro_batch * world_size)`，并断言它能整除。
4. 每个 micro batch 执行：`loss = criterion(model(x), y) / accum_steps; loss.backward()`。
5. 对非最后一个 micro，进入 `model.no_sync()`，跳过 DDP 中的 gradient all-reduce。
6. 最后一个 micro batch 之后，只运行一次 `optimizer.step()`。下一个窗口前清零 gradients。
7. Optimizer state 每个 effective batch 前进一次，learning rate schedule 也每个 effective batch tick 一次。

## Logging

每个 effective step 发出一条小型 JSON 记录，包含 `samples_per_sec`、`median_step_ms`、`sync_calls`、`accum_steps`、`effective_batch`。没有它，成本取舍就是不可见的。

## Failure modes

- 忘记 `/ accum_steps` 缩放：gradients 会放大 N 倍。
- 在窗口中途 step：parameters 会漂移。
- 每个 micro batch 都同步：网络受限，却没有统计收益。
- 与 mixed precision unscaling 混用：只缩放未缩放的 loss。
