# 生产量化：AWQ、GPTQ、GGUF K-quants、FP8、MXFP4/NVFP4

> 量化 format is not a universal choice ： it is a function of hardware, serving engine, 和 工作负载. GGUF Q4_K_M or Q5_K_M owns CPU and edge, delivered through llama.cpp and Ollama. GPTQ wins inside vLLM when you need multi-LoRA on the same base. AWQ with Marlin-AWQ kernels delivers ~741 tok/s on a 7B class 模型 with the best Pass@1 at INT4 ： the 2026 默认 for datacenter 生产化. FP8 stays the middle ground on Hopper, Ada, and Blackwell ： near-lossless and widely supported. NVFP4 and MXFP4 (Blackwell microscaling) are aggressive and require per-block validation. Two traps bite 团队: 校准 dataset must match 部署 domain, and KV 缓存 is separate from weight 量化 ： the AWQ lesson "my 模型 is 4 GB now" forgets the 10-30 GB KV 缓存 at 生产化 批处理 sizes.

**Type:** Learn
**Languages:** Python（标准库， 玩具 memory and throughput comparison across formats)
**Prerequisites:** Phase 10 · 13 (量化基础), Phase 17 · 04 (vLLM Serving Internals)
**Time:** ~75 分钟

## 学习目标

- Name the six 生产化 量化 formats and their sweet spots in 2026.
- Pick a format given hardware (CPU 对比 GPU, Hopper 对比 Blackwell), engine (vLLM, TRT-LLM, llama.cpp), 和 工作负载 (routine chat, 推理, multi-LoRA).
- Compute the weight 内存 saved and the KV 缓存 left untouched for a chosen format.
- Name 这个校准-dataset pitfall that degrades quantized 模型 on domain 流量.

## 问题

量化 reduces 内存 and HBM 带宽, which is exactly what decode needs. An FP16 70B 模型 is 140 GB of weights. Quantize weights to INT4 (AWQ or GPTQ) and 这个模型 is 35 GB ： fits in one H100 with room for KV 缓存, which matters because at 128 concurrent sequences with 2k context, KV 缓存 alone is 20-30 GB.

But 量化 is not free. Aggressive 量化 degrades 质量, especially on 推理-heavy tasks. Different formats work with different engines. Different hardware supports different precisions natively. The 2026 format zoo is real and you cannot copy someone else's choice ： you have to pick based on your stack.

## 概念

### 六种格式

| Format | Bits | Sweet spot | Engines |
|--------|------|-----------|---------|
| GGUF Q4_K_M / Q5_K_M | 4-5 | CPU, edge, laptops | llama.cpp, Ollama |
| GPTQ | 4-8 | Multi-LoRA on vLLM | vLLM, TGI |
| AWQ | 4 | Datacenter GPU 生产化 | vLLM (Marlin-AWQ), TGI |
| FP8 | 8 | Hopper/Ada/Blackwell datacenter | vLLM, TRT-LLM, SGLang |
| MXFP4 | 4 | Blackwell multi-user | TRT-LLM |
| NVFP4 | 4 | Blackwell multi-user | TRT-LLM |

### GGUF ： the CPU/edge 默认

GGUF is a file format, not 一个量化 scheme per se ： it bundles K-quant variants (Q2_K, Q3_K_M, Q4_K_M, Q5_K_M, Q6_K, Q8_0) in one container. Q4_K_M and Q5_K_M are 这个生产化 defaults ： near-BF16 质量 at 4-5 bits. Best choice for CPU or edge serving because llama.cpp is by far the fastest CPU inference engine.

吞吐量 penalty in vLLM: ~93 tok/s on 7B ： the format is not optimized for GPU kernels. Use GGUF when 这个部署 target is CPU/edge. Not otherwise.

### GPTQ ： multi-LoRA in vLLM

GPTQ is a post-training 量化 algorithm with 一个校准 pass. Marlin kernels make it fast on GPU (2.6x speedup 对比 non-Marlin GPTQ). ~712 tok/s on 7B.

The unique win: GPTQ-Int4 supports LoRA adapters in vLLM. If you are serving a base 模型 plus 10-50 fine-tuned variants (each as a LoRA), GPTQ is your path. NVFP4 does not support LoRA yet as of early 2026.

### AWQ ： the datacenter GPU 默认

Activation-aware Weight 量化. Protects 这个~1% most-salient weights during 量化. Marlin-AWQ kernels: 10.9x speedup 对比 naive. ~741 tok/s on 7B, best Pass@1 among INT4 formats.

选择AWQ for new GPU serving unless you need multi-LoRA (GPTQ) or aggressive Blackwell FP4 (NVFP4).

### FP8 ： the reliable middle

8-bit floating point. Near-lossless. Widely supported. Hopper Tensor Cores accelerate FP8 natively. Blackwell inherits. FP8 is the safe 2026 默认 什么时候 质量 is non-negotiable (推理, medical, code-gen). 内存 节省 are half of INT4 but 质量 风险 is far lower.

### MXFP4 / NVFP4 ： Blackwell aggressive

Microscaling FP4. Each block of weights has its own scale factor. Aggressive but hardware-accelerated on Blackwell Tensor Cores. Halve the bytes per 词元 versus FP8 ： the economic win in Phase 17 · 07.

Caveats:
- No LoRA support yet (early 2026).
- 质量 drop visible on 推理-heavy 工作负载.
- Validate on your eval set per 模型.

### 校准陷阱

AWQ and GPTQ require 一个校准 dataset ： typically C4 or WikiText. For domain 模型 (code, medical, legal), calibrating on generic web text lets the algorithm make wrong decisions about which weights to protect. Pass@1 on HumanEval can drop several points.

The fix: calibrate on in-domain data. Hundreds of domain samples is 通常 enough. Test on the eval set before shipping.

### KV 缓存陷阱

AWQ shrinks weights to 4 bits. KV 缓存 is separate and stays at FP16/FP8. For a 70B 模型 with AWQ:

- Weights: ~35 GB (INT4 from 140 GB).
- KV 缓存 at 128 concurrent × 2k context: ~20 GB.
- Activations: ~5 GB.
- Total: ~60 GB ： fits on H100 80GB.

Naively "I quantized my 模型 to 4 GB" forgets the other 30-50 GB. Budget HBM holistically.

Separately, KV 缓存 量化 (FP8 KV or INT8 KV) is a different choice with its own tradeoffs ： it affects attention accuracy 直接 and is not a free win.

### AWQ INT4 对推理任务有风险

Chain-of-thought, math, code-gen with long context ： these suffer visibly from aggressive 量化. AWQ INT4 loses ~3-5 points on MATH. For 推理-heavy 工作负载, ship FP8 or BF16; accept 这个内存 成本.

### 2026 年选择指南

- CPU/edge serve: GGUF Q4_K_M. Done.
- GPU serve, routine chat, no LoRA: AWQ.
- GPU serve, multi-LoRA: GPTQ with Marlin.
- 推理 工作负载: FP8.
- Blackwell datacenter, validated 质量: NVFP4 + FP8 KV.
- Ambiguous: run a 1,000-sample eval on each candidate format.

```figure
gpu-memory-breakdown
```

## 使用它

`code/main.py` computes 内存 footprint (weights + KV + activations) and relative 吞吐量 across the six formats for a range of 模型 sizes. Shows where KV 缓存 dominates, where weight compression pays, and where FP8 is the safe pick.

## 交付它

This lesson 产出 `outputs/skill-quantization-picker.md`. Given hardware, 模型 size, 工作负载 type, 和 质量 tolerance, picks a format and 产出 一个校准/validation 计划.

## 练习

1. Run `code/main.py`. For a 70B 模型 at 128 concurrent with 2k context, compute the total HBM for each format. Which format lets you fit on one H100 80GB?
2. You have a 7B coding 模型. Pick a format and 说明理由. If you were wrong about 质量 tolerance, what is the recovery path?
3. Compute 这个校准-dataset size 需要 to calibrate AWQ for a medical domain 模型. Why is more data not always better?
4. Read the Marlin-AWQ kernel paper or release notes. 解释 in three sentences why AWQ hits 741 tok/s on 7B while raw GPTQ hits ~712.
5. When does it make sense to combine AWQ weights with FP8 KV 缓存 对比 keeping KV at BF16?

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| GGUF | "llama.cpp format" | File format bundling K-quant variants; CPU/edge 默认 |
| Q4_K_M | "Q4 K M" | 4-bit K-quant medium; 这个生产化 GGUF 默认 |
| GPTQ | "gee pee tee q" | Post-train INT4 with 校准; supports LoRA in vLLM |
| AWQ | "a w q" | Activation-aware INT4; Marlin kernels; best Pass@1 at INT4 |
| Marlin kernels | "fast INT4 kernels" | Custom CUDA kernels for INT4 on Hopper; 10x speedup |
| FP8 | "eight-bit float" | Safe precision 默认 on Hopper/Ada/Blackwell |
| MXFP4 / NVFP4 | "microscaling four" | Blackwell 4-bit FP with per-block scale factors |
| 校准 dataset | "cal data" | 输入 text used to pick 量化 parameters; must match domain |
| KV 缓存 量化 | "KV INT8" | Separate choice from weights; affects attention accuracy |

## 延伸阅读

- [VRLA Tech — LLM Quantization 2026](https://vrlatech.com/llm-quantization-explained-int4-int8-fp8-awq-and-gptq-in-2026/) ： comparative 基准.
- [Jarvis Labs — vLLM Quantization Complete Guide](https://jarvislabs.ai/blog/vllm-quantization-complete-guide-benchmarks) ： 吞吐量 numbers by format.
- [PremAI — GGUF vs AWQ vs GPTQ vs bitsandbytes 2026](https://blog.premai.io/llm-quantization-guide-gguf-vs-awq-vs-gptq-vs-bitsandbytes-compared-2026/) ： format-by-format picking.
- [vLLM docs — Quantization](https://docs.vllm.ai/en/latest/features/quantization/index.html) ： supported formats and flags.
- [AWQ paper (arXiv:2306.00978)](https://arxiv.org/abs/2306.00978) ： original AWQ formulation.
- [GPTQ paper (arXiv:2210.17323)](https://arxiv.org/abs/2210.17323) ： original GPTQ formulation.
