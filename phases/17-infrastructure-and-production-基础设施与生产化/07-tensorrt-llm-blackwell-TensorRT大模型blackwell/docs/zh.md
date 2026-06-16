# 在 Blackwell 上使用 FP8 与 NVFP4 的 TensorRT-LLM

> TensorRT-LLM is NVIDIA-only but it wins on Blackwell. On GB200 NVL72 with Dynamo orchestration, SemiAnalysis InferenceX measured $0.012 per million 词元 on a 120B 模型 in Q1-Q2 2026, against $0.09/M on H100 + vLLM ： a 7x economic gap. The stack is three floating-point regimes compounded: FP8 stays 关键 for KV 缓存 and attention kernels because it has the dynamic range they need; NVFP4 (4-bit microscaling) handles weights and activations; multi-词元 prediction (MTP) and disaggregated prefill/decode add another 2-3x on top. Day-0 模型 support loads FP4 weights 直接 without post-training conversion. The catch for 2026 engineering 团队: TRT-LLM is a closed NVIDIA stack, so adopting it trades portability for 吞吐量. Run the math on your mix of 模型 and hardware before committing.

**Type:** Learn
**Languages:** Python（标准库， 玩具 FP8/NVFP4 memory and cost计算器)
**Prerequisites:** Phase 17 · 04 (vLLM Serving Internals), Phase 10 · 13 (Quantization)
**Time:** ~75 分钟

## 学习目标

- 解释 why FP8 stays 关键 for KV 缓存 and attention even when weights are in NVFP4.
- Compute the HBM footprint of a frontier 模型 under BF16, FP8, and NVFP4 and reason about where 这个节省 come from.
- Name the Blackwell-具体功能 TRT-LLM exploits (day-0 FP4, MTP, disaggregated serving, all-to-all primitives).
- Decide when TRT-LLM's NVIDIA-lock is worth the 7x 成本 gap 对比 vLLM on Hopper.

## 问题

The frontier of inference economics in 2026 is "how many 词元 per dollar". The answer depends on four stacked choices: hardware generation (Hopper H100/H200 对比 Blackwell B200/GB200), precision (BF16 → FP8 → NVFP4), serving engine (vLLM 对比 SGLang 对比 TRT-LLM), and orchestration (plain 对比 disaggregated 对比 Dynamo).

On Hopper with vLLM, a 120B MoE runs at ~$0.09 per million 词元. On Blackwell with TRT-LLM + Dynamo, the same 模型 runs at ~$0.012 ： 7x 更便宜. Some of that gap is hardware (Blackwell is 11-15x per-GPU LLM 吞吐量 对比 Hopper). Some is the stack: FP4 weights, MTP draft, disaggregated prefill/decode, and NVLink 5 all-to-all for MoE expert communication.

You cannot replicate this outside NVIDIA's stack. That is the tradeoff ： portability for economics. Understanding which stack choices give which share of the gap is the point of this lesson.

## 概念

### 为什么 FP8 仍是 KV 缓存底线

A common mistake in 2026: assuming NVFP4 applies everywhere. It does not. KV 缓存 needs FP8 (8-bit floating point) because it stores attention keys and values that span a wide dynamic range. Quantizing KV to FP4 causes catastrophic accuracy loss ： 这个尾部 of the distribution drops off and attention scores collapse. FP8's exponent bits give KV 缓存 the range it needs.

NVFP4 (2025-2026) applies to weights and activations. Microscaling: each block of weights has its own scale factor so small blocks can span different dynamic ranges without per-tensor scale loss. For activations, FP4 holds up because activations are small-range within a layer.

The typical Blackwell config:

- Weights: NVFP4 (4-bit microscaling).
- Activations: NVFP4.
- KV 缓存: FP8.
- Attention accumulator: FP32 (softmax stability).

### TRT-LLM 使用的 Blackwell 专用原语

- **Day-0 FP4 weights**: 模型 提供商 ship FP4 weights 直接; TRT-LLM loads without post-training conversion. No AWQ / GPTQ 步骤 for FP4.
- **Multi-词元 prediction (MTP)**: same idea as EAGLE (Phase 17 · 05) but integrated into the TRT-LLM build.
- **Disaggregated serving**: prefill and decode on separate GPU pools, KV 缓存 transferred over NVLink or InfiniBand. Same idea as Dynamo (Phase 17 · 20).
- **All-to-all communication primitives**: NVLink 5 cut MoE expert communication 延迟 by 3x 对比 Hopper. TRT-LLM's MoE kernels are tuned for this.
- **NVFP4 + MXFP8 microscaling**: hardware-accelerated scale-factor handling on Blackwell Tensor Cores.

### 你应该记住的数字

- HGX B200 at $0.02/M 词元 on GPT-OSS-120B via TRT-LLM.
- GB200 NVL72 at $0.012/M 词元 via Dynamo (orchestrating TRT-LLM).
- H100 + vLLM ≈ $0.09/M 词元 on comparable 工作负载.
- 2.8x 吞吐量 gain in three months of TRT-LLM updates (2026).
- 11-15x per-GPU LLM 吞吐量, Blackwell 对比 Hopper.
- MLPerf Inference v6.0 (April 2026): Blackwell dominates every submitted task.

### FP4 在质量上真正付出的代价

NVFP4 is aggressive. On 推理-heavy 工作负载 (chain-of-thought, math, code-gen with long context), FP4 weights degrade visibly. Per-block 校准 mitigates but does not eliminate. 团队 shipping 推理 模型 often use FP8 weights + FP4 activations as a compromise, or stick to H200 with FP8 throughout.

The rule: always validate task 质量 on your eval set before committing to NVFP4 weights.

### 为什么这是 NVIDIA 锁定决策

TRT-LLM is C++ + CUDA + closed-source kernels. 模型 need to be compiled for 一个具体 GPU SKU. No AMD, no Intel, no ARM. If your infra strategy is multi-vendor, TRT-LLM is a non-starter for the TRT-LLM-served tier ： you can still serve from vLLM on mixed hardware. If you are NVIDIA-only, the 7x gap pays for the lock.

### 2026 年实用配方

For 一个$100M+ annual inference bill, running on Hopper + vLLM leaves 7-10x on the table. Migrate 成本-dominant 工作负载 to Blackwell + TRT-LLM + Dynamo. Keep experimentation tier on H100 + vLLM for 模型 iteration speed. Validate 质量 on each NVFP4-converted 模型 before 生产化.

### 分离式架构的额外收益

TRT-LLM's disaggregated serving (separate prefill and decode pools) is covered in depth in Phase 17 · 20. On Blackwell, the multiplier stacks: FP4 weights × MTP speedup × disaggregated placement × cache-aware 路由. The 7x number assumes this full stack.

```figure
pipeline-parallel
```

## 使用它

`code/main.py` computes HBM footprint, decode 吞吐量 (内存-bound regime), 和 $/M-词元 for 一个模型 across three stacks: H100 + BF16 + vLLM, H100 + FP8 + vLLM, B200 + NVFP4/FP8 + TRT-LLM. Run it to see the compounding effect and the share of the gap each change contributes.

## 交付它

This lesson 产出 `outputs/skill-trtllm-blackwell-advisor.md`. Given 一个工作负载, 模型 size, and annual 词元 volume, it decides whether the Blackwell + TRT-LLM stack is worth the NVIDIA-lock.

## 练习

1. Run `code/main.py`. On a 120B MoE with 30% active parameters, compute 这个内存-带宽-limited decode 吞吐量 on H100 BF16, H100 FP8, and B200 NVFP4/FP8. Where does the biggest jump come from?
2. 一个客户 spends $2M/year on H100 + vLLM. What is 这个盈亏平衡 number of Blackwell GPUs they need to buy to amortize 一个迁移 to TRT-LLM in 12 months, given the 7x economic gap?
3. You see accuracy drop 3 points on MATH after NVFP4 weight conversion. Name two recovery paths: one 质量-first (keep FP8 weights), one 成本-first (calibrate with in-domain data).
4. Read the MLPerf v6.0 inference results. Which task has the smallest Blackwell-over-Hopper gap, and why?
5. Compute the HBM 需要 for a 405B 模型 at NVFP4 weights + FP8 KV 缓存 at 128k context. Does it fit on a single GB200 NVL72 node?

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| FP8 | "eight-bit float" | 8-bit floating point; used for KV 缓存 and attention due to dynamic range |
| NVFP4 | "four-bit micro" | NVIDIA's 4-bit microscaling FP format; weights and activations on Blackwell |
| MXFP8 | "MX eight" | Microscaling FP8 variant; hardware-accelerated on Blackwell Tensor Cores |
| Day-0 FP4 | "ship FP4 weights" | 模型 提供商 release weights already in FP4; no post-train conversion 步骤 |
| MTP | "multi-词元 prediction" | TRT-LLM's integrated speculative-decoding draft (Phase 17 · 05) |
| Disaggregated serving | "split prefill/decode" | Prefill and decode on separate GPU pools; KV transferred over NVLink/IB |
| All-to-all | "MoE expert comm" | Communication 模式 路由 词元 to expert GPUs; NVLink 5 cuts 3x |
| InferenceX | "SemiAnalysis inference bench" | The 2026 industry-accepted 成本-per-词元 基准 |

## 延伸阅读

- [NVIDIA — Blackwell Ultra MLPerf Inference v6.0](https://developer.nvidia.com/blog/nvidia-blackwell-ultra-sets-new-inference-records-in-mlperf-debut/) ： April 2026 MLPerf results.
- [NVIDIA — MoE Inference on Blackwell](https://developer.nvidia.com/blog/delivering-massive-performance-leaps-for-mixture-of-experts-inference-on-nvidia-blackwell/) ： NVLink 5 all-to-all and MoE kernels.
- [TensorRT-LLM Overview](https://nvidia.github.io/TensorRT-LLM/overview.html) ： official engine documentation.
- [NVIDIA — Introducing Dynamo](https://developer.nvidia.com/blog/introducing-nvidia-dynamo-a-low-latency-distributed-inference-framework-for-scaling-reasoning-ai-models/) ： disaggregated orchestration above TRT-LLM.
- [MLPerf Inference](https://mlcommons.org/benchmarks/inference-datacenter/) ： 这个基准 suite that publishes Blackwell numbers.
