---
name: skill-quantization-zh
description: Choose the right 量化 strategy for deploying LLMs based on hardware, 质量, and 延迟 constraints
version: 1.0.0
phase: 10
lesson: 11
tags: [quantization, inference, deployment, optimization, fp8, int4, int8, gptq, awq, gguf]
---

# 量化 Decision Framework

当deploying a 语言模型, use this framework to select the right number format, 量化 method, and 质量 验证 strategy.

## 输入 Requirements

Provide:
- **模型** (name, 参数 count, original precision)
- **目标 hardware** (GPU 模型/VRAM, CPU, Apple Silicon, 边 device)
- **延迟 目标** (词元/second, time to first 词元)
- **质量 floor** (max acceptable perplexity increase, 基准 delta)
- **Serving pattern** (批次 size, max 上下文 length, concurrent users)

## Quick Selection

|Your Situation|Format|Method|Expected 质量 损失|
|---------------|--------|--------|----------------------|
|H100 GPU, maximum throughput|FP8 E4M3|Native H100 casting|< 0.1%|
|A100/A10, need 2x throughput|INT8|LLM.int8() or SmoothQuant|< 0.5%|
|Single 24GB GPU, 70B 模型|INT4|AWQ or GPTQ|1-3%|
|MacBook / Apple Silicon|INT4 GGUF|Q4_K_M via llama.cpp|1-2%|
|Mobile / 边 device|INT4 or INT3|QAT + device-specific|2-5%|
|Maximum 压缩, some 损失 OK|INT2|QuIP# or AQLM|5-15%|
|训练 (mixed precision)|BF16 + FP32 accum|Native framework support|0%|

## Precision Selection by Component

Not all tensors should get the same treatment.

|Component|Safe Minimum|Recommended|Avoid|
|-----------|-------------|-------------|-------|
|FFN 权重|INT4|INT4 (AWQ/GPTQ)|INT2 without QAT|
|注意力 权重|INT4|INT8 or FP8|INT2|
|嵌入 层|INT8|FP16 (keep original)|INT4|
|输出 头|INT8|FP16 (keep original)|INT4|
|KV 缓存|FP8|FP8 or INT8|INT4 at long 上下文|
|注意力 logits|FP16|FP16 or BF16|INT8|
|Activations (推理)|INT8|FP8 or INT8|INT4|

## Method Comparison

### GPTQ
- **When:** GPU 推理, you want a Hugging Face-compatible 模型
- **Calibration 数据:** 128 examples, 2048 词元 each
- **时间:** 30-60 分钟 for 70B on A100
- **Tooling:** `auto-gptq`, `exllama`, `exllamav2`
- **Strength:** Well-tested, huge 模型 zoo on Hugging Face
- **Weakness:** Slower than AWQ to apply, slightly lower 质量 than AWQ on some 模型

### AWQ
- **When:** GPU 推理, you want best quality-per-bit
- **Calibration 数据:** 128 examples
- **时间:** 15-30 分钟 for 70B on A100
- **Tooling:** `autoawq`, `vLLM` (native support)
- **Strength:** Best INT4 质量, fast to apply, vLLM integration
- **Weakness:** Smaller 模型 zoo than GPTQ

### GGUF
- **When:** CPU 推理, Apple Silicon, llama.cpp ecosystem
- **变体s:** Q2_K, Q3_K_S/M/L, Q4_K_S/M, Q5_K_S/M, Q6_K, Q8_0, F16
- **Recommended default:** Q4_K_M (best 质量/size balance)
- **Tooling:** `llama.cpp`, `ollama`, `LM Studio`
- **Strength:** Self-contained files, mixed precision, massive ecosystem
- **Weakness:** Not optimal for GPU (designed for CPU/Metal)

### SmoothQuant
- **When:** INT8 on GPU, need both 权重 and 激活 量化
- **Key idea:** Migrate 量化 difficulty from activations to 权重 via per-channel 扩展
- **Tooling:** `smoothquant`, `TensorRT-LLM`
- **Strength:** Enables W8A8 (both 权重 and activations in INT8) for 2x speedup
- **Weakness:** INT8 only, does not extend to INT4

## 质量 验证 协议

After quantizing, 验证 before deploying:

1. **Perplexity test.** 计算 on WikiText-2 or your 领域 语料库. Delta < 0.5 is excellent, 0.5-1.0 is good, > 2.0 is a problem.

2. **基准 sweep.** Run MMLU (general), GSM8K (math), HumanEval (code). Math and code are most sensitive to precision 损失.

3. **输出 comparison.** 生成 100 响应 from both original and 量化的 模型. Use LLM-as-judge to 计算 win 速率. 目标: 量化的 模型 wins or ties on > 90% of prompts.

4. **延迟 measurement.** Measure 词元/second at 批次 size 1 and your 目标 批次 size. Verify the speedup justifies the 质量 成本.

5. **Long-context test.** If serving long contexts (> 4K 词元), test at your maximum 上下文 length. KV 缓存 量化 错误 compound with 序列 length.

## 内存 预算 Calculator

```text
Weight memory (GB) = parameters (B) * bits / 8 / 1.073741824
KV cache per token (MB) = 2 * num_layers * d_model * bits / 8 / 1048576
KV cache for context (GB) = kv_per_token * max_context_length / 1024
Activation memory (GB) ~ 1-4 GB (relatively constant, depends on batch size)
Total = weight_memory + kv_cache + activation_memory + overhead (10-20%)
```

Example for Llama 3 70B at INT4, 32K 上下文:
- 权重: 70B * 4 / 8 / 1.07 = 32.6 GB
- KV 缓存 (FP16): 2 * 80 * 8192 * 16 / 8 / 1e9 * 32768 = ~40 GB
- KV 缓存 (FP8): ~20 GB
- Total with FP8 KV: ~55 GB (fits one 80GB A100)

## Common Mistakes

|Mistake|Why It Fails|修复|
|---------|-------------|-----|
|Quantizing the 嵌入 层 to INT4|First 层 amplifies 错误 through entire 模型|Keep 嵌入s at FP16 or INT8|
|Using per-tensor scales for INT4|One outlier row destroys precision for all rows|Use per-channel or per-group scales|
|Not calibrating GPTQ/AWQ|规模 factors are wrong without representative 数据|Use 128 examples from your 领域|
|Same bit-width for all 层|First/last 层 are more sensitive|Mixed precision: higher bits for first/last|
|Quantizing KV 缓存 at very long 上下文|错误 compound quadratically with 序列 length|Use FP8 for KV 缓存, not INT4|
|Skipping 质量 验证|Some 模型 quantize poorly (especially at boundaries)|Always run perplexity + 任务 evals|

## Deployment Recipes

### Recipe 1: vLLM with AWQ (GPU 服务器)
```text
pip install vllm autoawq
vllm serve model-awq --quantization awq --dtype half --max-model-len 8192
```

### Recipe 2: llama.cpp with GGUF (MacBook)
```text
./llama-server -m model.Q4_K_M.gguf -c 4096 -ngl 99
```

### Recipe 3: TensorRT-LLM with FP8 (H100)
```text
trtllm-build --model_dir model --output_dir engine --dtype float16 --use_fp8
```
