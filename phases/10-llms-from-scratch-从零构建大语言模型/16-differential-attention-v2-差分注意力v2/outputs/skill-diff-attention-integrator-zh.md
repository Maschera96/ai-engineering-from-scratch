---
name: diff-attention-integrator-zh
description: Integration plan for adding Differential 注意力 V2 to a new 预训练 run or LoRA fine-tune.
version: 1.0.0
phase: 10
lesson: 16
tags: [differential-attention, diff-transformer, long-context, flash-attention, pre-training, lora]
---

给定a 模型 架构 (隐藏, 头, KV 头, 层, d_head), a 目标 上下文 length, a 幻觉 or long-context profile (failure modes on your existing evals), and a 训练 预算 (词元 available, GPU-hours), produce an integration plan for DIFF V2.

Produce:

1. Integration mode. From-scratch 预训练, mid-training 架构 swap, or LoRA fine-tune on Q projections. Justify the choice against the 训练 预算 and the existing 权重 available.
2. 架构 diff. Concrete field-by-field change list: which projections grow, which stay the same, which 参数 count you are adding, and where the subtraction gets placed in the 注意力 块. Include `lambda_init` 调度 by 层 深度 (`0.8 - 0.6 * exp(-0.3 * (depth - 1))` is the paper's default; adjust per-depth if layerwise telemetry shows instability).
3. Kernel choice. Confirm FlashAttention 2 or 3 support given V2's head-count doubling. Reject V1's custom-kernel path unless the 用户 explicitly needs it for reproducibility.
4. 内存 预算. KV 缓存 stays at 基线 (KV 头 unchanged). 计算 per-token 激活 内存 delta (extra Q 头, extra 计算). Report absolute numbers at the 目标 上下文.
5. 训练 stability plan. Describe what to monitor: `lambda` drift per 层, 注意力 熵 per 头, 梯度 方差 on the Q projections. Name the specific 指标 that should trigger a rollback to 基线 注意力 if telemetry indicates divergence.

Hard rejects:
- Adding DIFF 注意力 to a pre-trained 模型 without continued 预训练. 输出 distributions drift — not a drop-in fix.
- DIFF V1 for any new run past April 2026. V2 is strictly better in all measured 维度.
- Integrating DIFF without also enabling long-context 训练 数据. The benefit only shows past 32k.
- Changing `lambda_init` to a negative value without a controlled experiment. Negative init subtracts more than the 噪声 floor and collapses 训练.

Refusal rules:
- 如果the 目标 上下文 is below 16k, refuse the integration and recommend standard 注意力. The added 参数 成本 is not justified by the noise-floor argument.
- 如果the 用户 cannot provide long-context 评估 数据 (RULER, needle-in-haystack, MultiNeedle), refuse and request calibration 数据 first.
- 如果the 用户 is on a pre-FlashAttention-2 stack, refuse and recommend upgrading the stack before attempting integration.

输出： a one-page integration plan listing mode, param count delta, KV 缓存 impact, FlashAttention confirmation, `lambda` 调度, and a 3-指标 monitoring board. End with a "success criterion" paragraph naming the specific long-context 评估 number (percentage point delta on RULER 64k or equivalent) that would justify keeping DIFF V2 in the 架构 versus reverting.
