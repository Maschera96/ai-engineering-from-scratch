---
name: quantization-picker-zh
description: 选择a 2026 量化 format given hardware, engine, 工作负载, 和 质量 tolerance, and produce 一个校准 + validation 计划.
version: 1.0.0
phase: 17
lesson: 09
tags: [quantization, awq, gptq, gguf, fp8, nvfp4, calibration]
---

给定hardware (CPU / H100 / H200 / B200 / GB200, with count), engine (llama.cpp / vLLM / TRT-LLM / SGLang), 模型 (size + task type ： routine chat / 推理 / code / multi-LoRA), 和 质量 tolerance (can absorb N-point drop on HumanEval / MATH / MMLU), pick 一个量化 format and produce a validation 计划.

产出：

1. Format 建议. One of: GGUF Q4_K_M, GGUF Q5_K_M, GPTQ-Int4 + Marlin, AWQ-Int4 + Marlin, FP8, NVFP4 + FP8 KV, or a stacked combo. 说明理由 by 这个决策 tree: CPU → GGUF; 推理 → FP8; multi-LoRA on vLLM → GPTQ; routine GPU chat → AWQ; Blackwell validated → NVFP4.
2. 内存 budget. Report weights + KV 缓存 (at reported concurrency × context) + activations. Confirm it fits on the target GPU or call out multi-GPU requirement.
3. 校准 计划. Dataset source (domain-matched for AWQ/GPTQ; generic C4/WikiText as a last resort). Sample count (500-2000 for domain). Validation set (10% held out from 校准 pool).
4. Validation 计划. Eval set matched to task: HumanEval for code, MATH/MMLU for 推理, MT-Bench for chat. Baseline BF16 对比 quantized. Ship if drop ≤ 质量 tolerance.
5. KV 缓存 决策. Separate from weight 量化. Recommend FP8 KV for 推理; BF16 KV if attention accuracy is marginal; INT8 KV only after validation.
6. 回滚 path. Keep BF16/FP8 weights on disk; flag to switch back if 生产化 质量 degrades.

硬性拒绝：
- Recommending NVFP4 weights on 推理-heavy 工作负载 without eval-set validation.
- Calibrating on generic web data for domain 模型. Always use in-domain.
- Forgetting the KV 缓存 in HBM budget. Always itemize.
- Claiming 吞吐量 numbers without naming the kernels (Marlin-AWQ 对比 plain AWQ is 10x).

拒绝规则：
- If 这个工作负载 is inherently 质量-marginal (open-ended creative generation, edge-case 推理), 拒绝 aggressive INT4. Stay FP8 or BF16.
- If the engine is llama.cpp, 拒绝 any format other than GGUF. Matching format to engine is table stakes.
- If the user cannot run a 1,000-sample eval, 拒绝. No blind 量化 in 生产化.

输出： a one-page 量化 pick listing chosen format, HBM budget, 校准 计划, validation 计划, KV 缓存 决策, 和 回滚 path. End with 一个"what to measure next" paragraph naming one of eval-set delta, KV 缓存 pressure under peak concurrency, 或 吞吐量 at real 批处理 size depending on the key 风险.
