---
name: mtp-planner-zh
description: Plan a multi-token 预测 integration for a new 预训练 run.
version: 1.0.0
phase: 10
lesson: 18
tags: [mtp, multi-token-prediction, deepseek-v3, pre-training, speculative-decoding]
---

给定a 预训练 run specification (模型 规模, 隐藏 size, 层, 数据 词元 预算, GPU 拓扑, 目标 deployment) and a stated goal (denser 训练 信号 vs speculative-decoding draft vs both), produce an MTP integration plan.

Produce:

1. 深度 D. Pick 1 or 2. DeepSeek-V3 uses D=1 and reports the first-depth speculative-decoding acceptance at 80%+. D=2 is diminishing-returns territory for most runs. Justify the choice against 计算 预算 — each extra 深度 adds roughly one transformer 块 of 计算 per 训练 步骤.
2. Lambda 调度. Default: 0.3 for the first 10% of 训练, 0.1 afterward. Adjust up to 0.5 early for small 模型 (under 7B) where the denser 信号 matters more; adjust down if you observe the MTP 损失 dominating the main 损失.
3. 参数 预算. Report per-module 参数 count against the main 模型. Confirm overhead is under 5% of main 参数 (稠密) or under 3% (MoE).
4. 内存 and 计算 overhead. Quantify extra forward-pass FLOPs per 步骤 (roughly `D * transformer_block_cost`), extra backward-pass 内存 (激活 内存 for D modules), and extra peak VRAM (shared 嵌入 and 头 do not count, projection and transformer 块 do).
5. 推理-time wiring. Describe how to consume the MTP module as a speculative-decoding draft at 推理. Name the Leviathan rule integration path and the KV-rollback bookkeeping. Confirm compatibility with the 目标 推理 stack (vLLM, SGLang, TensorRT-LLM).

Hard rejects:
- Adding MTP to a 稠密 模型 pre-trained without it. Cannot retrofit — the MTP modules are not 训练后的.
- D > 2 for a first integration. Gain over D=1 is small; complexity grows quickly.
- MTP on a 模型 under 1B active 参数. 信号 is weaker than the overhead 成本 at that 规模.
- Using 并行 (Gloeckle-style) 头 when the goal is 推测解码. They do not 链 causally.

Refusal rules:
- 如果the 预训练 数据 is dominated by short sequences (under 2k), refuse. MTP gains assume sequences long enough for depth-2 supervision to matter.
- 如果the 目标 推理 stack does not support 推测解码 at all, note that MTP still buys the denser 训练 信号 and proceed, but flag the mismatch.
- 如果the 用户 is continuing 预训练 on an existing 稠密 checkpoint without MTP, refuse and recommend adding MTP only at the start of a clean 训练 run or at a clean data-boundary reset.

输出： a one-page integration plan listing D, lambda 调度, 参数 overhead (absolute and percentage), 计算 overhead (percentage per 训练 步骤), and the 推理-time speculative-decoding wiring plan. End with a "success criterion" paragraph naming the measured 指标 that justifies keeping MTP: acceptance 速率 at 深度 1 after 50B 训练 词元 must be above 70%, otherwise the 架构 should be reverted.
