---
name: nsa-integrator-zh
description: Integration plan for Native 稀疏 注意力 in a long-context 预训练 run.
version: 1.0.0
phase: 10
lesson: 17
tags: [nsa, sparse-attention, long-context, pre-training, kernel-aligned, deepseek]
---

给定a long-context 预训练 run specification (目标 上下文, base 架构, 训练 词元 available, GPU 拓扑, deployment 目标), produce an NSA integration plan.

Produce:

1. 压缩 块 size `l`. Pick 32, 64, or 128. Justify against 目标 上下文: `l = 32` for 16k-32k, `l = 64` for 64k-128k, `l = 128` for 256k-plus. Larger `l` means fewer compressed keys but coarser 路由 信号.
2. Top-k selection count. Pick between 8 and 32. The paper's default is 16. Justify against the 目标 任务 mix: reasoning-heavy tasks (math, code) benefit from higher `k` because selection precision matters more. Retrieval-heavy tasks work at lower `k`.
3. Sliding window `W`. Pick 256, 512, or 1024. Default 512. Shorter for heavily 结构化 content (code) where local 上下文 is enough; longer for prose.
4. Gate MLP. Specify 宽度 and initialization. Default: linear 层 from `hidden` to 3, with `sigmoid` or `softplus` 激活. Warn if gate 权重 崩塌 to favor one branch — this indicates `l`, `k`, or `W` is mistuned.
5. Kernel choice. Confirm Triton or CUDA kernel availability for the 目标 accelerator. Reject 备选方案 to 稠密 注意力 at 推理 (the whole point of NSA is to save decode 计算). If only forward kernels exist and not backward, refuse 预训练 and recommend continued 训练 on existing 稠密 checkpoints.

Hard rejects:
- NSA on a 模型 pre-trained with 稠密 注意力 without continued 预训练. Cannot be bolted on at 推理.
- 目标 上下文 under 16k. The three-branch overhead dominates.
- 推理-only deployments on stacks without NSA kernel support. Recommend MLA or sliding-window 注意力 instead.

Refusal rules:
- 如果long-context 评估 数据 (RULER, LongBench, needle-in-haystack) is not available, refuse and request calibration 数据 first.
- 如果the training-data 上下文 分布 is dominated by short sequences, refuse and recommend 数据 reweighting before integrating NSA.
- 如果the accelerator is older than A100, refuse — NSA's kernel advantages assume H100/H200/MI300 内存 hierarchies.

输出： a one-page integration plan listing `l`, `k`, `W`, gate 配置, kernel path, and expected 计算 savings at 目标 上下文. End with a "success criterion" paragraph: the specific RULER or LongBench number (percentage points vs a matched dense-attention 基线) that justifies keeping NSA. Include a rollback trigger — the 指标 阈值 below which the 架构 should be reverted to MLA or 稠密 GQA.
