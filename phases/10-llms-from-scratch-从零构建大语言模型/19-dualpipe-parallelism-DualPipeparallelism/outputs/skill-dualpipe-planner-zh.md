---
name: dualpipe-planner-zh
description: Plan a 流水线 parallelism strategy (1F1B, Zero Bubble, DualPipe, DualPipeV) for a 训练 cluster.
version: 1.0.0
phase: 10
lesson: 19
tags: [pipeline-parallelism, dualpipe, dualpipev, zero-bubble, expert-parallelism, distributed-training]
---

给定a 训练 cluster specification (total GPU count, interconnect 拓扑, accelerator 模型, 内存 per GPU), a 模型 shape (total params, active params, MoE or 稠密, expected 层 count), and a 目标 training-data volume, recommend a 流水线 parallelism strategy and confirm the expected bubble fraction.

Produce:

1. 流水线 深度 P. Pick based on GPU 内存 预算 (must fit one 流水线 stage per 排序), MoE vs 稠密, and interconnect bandwidth. Range: 4 for small clusters, 16-32 for frontier MoE 训练.
2. Micro-batch count M. Must be divisible by 2 for DualPipe and DualPipeV. Typical 比例 M/P between 8 and 16. Justify against gradient-accumulation 目标 and 激活 内存 at the 目标 序列 length.
3. 调度 choice. Pick from 1F1B, Zero Bubble, DualPipe, DualPipeV. Decision table: 稠密 训练 under 500 GPUs -> Zero Bubble. MoE with expert parallelism -> DualPipe. 稠密 训练 above 500 GPUs without heavy all-to-all -> DualPipeV. Small runs under 100 GPUs -> 1F1B is fine.
4. Expected bubble fraction. 计算 for the chosen 调度 at the 目标 P and M. Report as percentage and as absolute GPU-hours saved versus 1F1B at the total 训练 预算.
5. 参数 replication plan (DualPipe only). Confirm the 2x 参数 replication fits in available VRAM. Report the effective 参数 密度 per GPU given the chosen P.

Hard rejects:
- DualPipe without Expert Parallelism. The 2x replication is not justified without EP-heavy comms to hide.
- P > 64 on any 训练 run. Bubble fraction grows linearly with P regardless of 调度.
- Micro-batch count not divisible by 2 for DualPipe/DualPipeV. The 调度 will not close.
- 流水线 parallelism at all when the 模型 fits in one GPU's 内存. Use 数据 parallelism only.

Refusal rules:
- 如果the interconnect is 200Gbps or slower per GPU, refuse DualPipe and recommend DualPipeV. The all-to-all overlap window is too narrow to justify the replication.
- 如果the 用户 cannot provide a custom all-to-all kernel suitable for their cluster 拓扑, recommend Zero Bubble rather than DualPipe.
- 如果the 训练 run is below 1B 词元, refuse 流水线 parallelism planning entirely and recommend 数据 parallelism plus tensor parallelism.

输出： a one-page plan listing P, M, 调度, expected bubble fraction, 参数 replication 成本 (if DualPipe), and an all-to-all kernel recommendation. End with a "rollback trigger" paragraph naming the specific utilization 指标 (aggregate GPU utilization percentage, measured over the first 1000 步骤) that would justify switching to a simpler 调度 if the 目标 number is not hit.
