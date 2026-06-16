---
name: checkpointing-planner-zh
description: Choose an 激活 recomputation 策略 per 层 (none / selective / full / offload) given a 训练 配置 and HBM 预算.
version: 1.0.0
phase: 10
lesson: 34
tags: [gradient-checkpointing, activation-recomputation, selective-checkpoint, fsdp-offload, training-memory]
---

给定the 训练 配置 (层 count L, 隐藏 size d, 序列 length S, microbatch B, dtype bytes per value, 注意力 kernel, tensor-parallel degree TP, pipeline-parallel degree PP, expert-parallel degree EP if MoE) and the per-rank HBM 预算 after 权重 and 优化器 状态, 输出：

1. Per-layer 策略. For each 层 family in the stack (嵌入, 注意力, FFN, MoE expert, norm, 输出 头) pick none, selective, full, or offload. Default to selective for 注意力 when S exceeds 4_096; default to none on residual streams and norms; default to offload on FFN only when the measured PCIe transfer time for that 层's activations is less than its measured recompute time.
2. Segment size k. If full checkpointing is on, pick k as round(sqrt(L)) for uniform 层 成本, smaller k when 激活 内存 dominates the 预算. Report extra FLOP percentage as (1/k) of forward FLOPs.
3. FlashAttention interaction. Confirm whether the 注意力 kernel already recomputes softmax. If yes, selective 注意力 checkpointing buys little; downgrade to none. 状态 the kernel by name (FlashAttention-2/3, xFormers memory-efficient, vanilla).
4. TP / PP plan. For TP, name the activations that need gather or rescatter on recompute and the per-step communication bytes added. For PP, confirm which 流水线 stages get checkpointed end-to-end so reverse microbatches free 激活 内存 before flowing back.
5. 预算 math. 预测 激活 内存 before and after the 策略 (in MB per 排序). 预测 FLOP overhead as percent of fwd+bwd. Reject any plan that does not fit in the HBM 预算 with 10 percent headroom.

拒绝full checkpointing every 层 when selective on 注意力 alone closes the 预算; profile shows the FLOP overhead is many times higher than selective for the same 内存 savings, and the exact 比例 is workload-specific. Refuse offload when the 层's measured 激活 transfer time on the 目标 PCIe link exceeds its measured recompute time; recompute wins. Refuse "checkpoint everywhere" for FP8 训练 when the chosen framework does not snapshot amax history; the recompute will drift the 规模 and silently corrupt gradients.

Example 输入: "L=64, d=8192, S=8192, B=1, bf16, FlashAttention-3, TP=8, PP=4, HBM 预算 per 排序 32 GB after 权重, MoE with 8 experts and EP=8."

Example 输出：
- Per-layer 策略: 注意力 selective, FFN none, MoE expert full, 嵌入 none, 输出 头 offload.
- Segment size: full applied on MoE only at k=8; FLOP overhead 12 percent on expert path, 0 elsewhere.
- FlashAttention interaction: FA-3 already recomputes softmax; selective at the 层 wrapper, not inside the kernel.
- TP / PP plan: TP gather of the 注意力 输入 on recompute, 0.3 GB per 步骤 extra comms; PP stages each checkpoint their full forward; PP stage 3 retains its activations for the final backward.
- 预算 math: activations 38 GB without 策略, 11 GB with 策略. Total FLOP overhead 7.5 percent fwd+bwd.
