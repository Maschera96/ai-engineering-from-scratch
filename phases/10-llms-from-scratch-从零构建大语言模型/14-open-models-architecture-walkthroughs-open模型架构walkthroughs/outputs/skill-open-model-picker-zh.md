---
name: open-model-picker-zh
description: 选择an 开放 LLM family, 量化, and 推理 stack for a given deployment 目标.
version: 1.0.0
phase: 10
lesson: 14
tags: [open-models, llama, deepseek, mixtral, qwen, gemma, moe, gqa, mla, quantization]
---

给定a deployment 目标 (GPU type, VRAM per GPU, number of GPUs, 目标 上下文 length, 目标 p50/p99 延迟, peak concurrent requests) and a 任务 profile (chat, code, 推理, long-context 检索, 工具使用), recommend an 开放 模型 plus serving stack with explicit 推理 about each of the six architectural knobs from Lesson 14.

Produce:

1. 模型 shortlist. Three candidates, each with total params, active params (MoE-aware), 架构 flags (norm / 激活 / position / 注意力 / MoE / 上下文), and the single 原因 it made the shortlist.
2. 内存 预算 check. For the top candidate: 权重 内存 at BF16 and at the chosen 量化; KV 缓存 at 目标 上下文 for the 目标 批次 size; 激活 headroom. Halt the recommendation if 权重 + KV 缓存 + activations exceed available VRAM.
3. 量化 choice. GPTQ-4bit, AWQ-4bit, FP8, or BF16. Justify against accuracy sensitivity of the 任务 (code / math / 推理 tasks take a bigger hit from aggressive 量化 than chat or 检索).
4. 推理 stack. vLLM, TensorRT-LLM, SGLang, or llama.cpp. Justify against: continuous batching need, 推测解码 support, 量化 format compatibility, and single-node vs multi-node 拓扑.
5. Throughput sanity check. Prefill 词元/sec and decode 词元/sec estimates based on GPU 内存 bandwidth (decode) and TFLOPs (prefill). Reject the recommendation if decode throughput is below the 目标's concurrent-user floor.
6. 备选方案. Second choice if the top candidate exceeds VRAM or throughput 预算. Always name one.

Hard rejects:
- 稠密 模型 above 30B on a single 24GB consumer GPU without offloading or aggressive 量化.
- MoE 模型 on a serving stack without expert-parallel support.
- Long-context (128k+) on architectures without GQA or MLA (KV 缓存 explodes).
- Any recommendation that does not name the specific 模型 revision (e.g., "Llama 3 8B Instruct v3.1", not "Llama 3").

输出： a one-page recommendation listing 模型, 量化, stack, with numbered evidence for each decision. End with a "worth reconsidering if..." paragraph naming the specific capability or deployment 参数 that would flip the choice.
