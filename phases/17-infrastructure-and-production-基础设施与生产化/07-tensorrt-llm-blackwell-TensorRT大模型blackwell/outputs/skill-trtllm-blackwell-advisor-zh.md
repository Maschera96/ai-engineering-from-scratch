---
name: trtllm-blackwell-advisor-zh
description: Decide whether Blackwell + TensorRT-LLM + Dynamo is worth the NVIDIA-lock for 一个给定工作负载 and budget.
version: 1.0.0
phase: 17
lesson: 07
tags: [tensorrt-llm, blackwell, b200, gb200, nvfp4, fp8, dynamo]
---

给定一个工作负载 (模型 size, active params, annual 词元 volume, 质量 sensitivity ： 推理-heavy or routine), current infra (H100/H200/B200 GPUs, serving engine), and budget, produce a Blackwell + TRT-LLM 迁移 advisory.

产出：

1. Current baseline. Compute current $/M 词元 and annual spend from reported volume and per-GPU-小时 定价. Flag if baseline is already on Blackwell + TRT-LLM.
2. Target stack. Recommend exact precision mix (weights: NVFP4 or FP8; KV 缓存: FP8; activations: NVFP4; accumulator: FP32). For 推理-heavy 工作负载, recommend FP8 weights first, NVFP4 only after per-block 校准 validated on the eval set.
3. Expected 节省. From the 2026 成本 shape: H100 + vLLM ~$0.09/M → B200 + TRT-LLM ~$0.02/M → GB200 NVL72 + Dynamo ~$0.012/M. Project annual 节省 for 这个工作负载's 词元 volume.
4. 迁移 成本. Engineering time (10-30 engineer-weeks for first 迁移). 质量-validation pass. GPU CapEx or rental commitment.
5. 盈亏平衡 horizon. Months of 生产化 需要 to amortize 迁移. If > 18 months, flag as marginal.
6. Lock-in 风险. TRT-LLM is NVIDIA-only. Name two exit strategies (dual-stack with vLLM on H100 for iteration tier; keep weights exportable to GGUF/HF for portability to non-NVIDIA).

硬性拒绝：
- Recommending NVFP4 weights on 推理-heavy 模型 without an eval-set validation 步骤.
- Claiming the 7x gap without naming 这个词元 volume the math assumes.
- Ignoring 质量 validation for FP4 weight conversion. Always run.

拒绝规则：
- If annual inference spend < $500K, 拒绝 迁移. The engineering 成本 does not amortize. Stay on vLLM + Hopper.
- If 这个团队 has any AMD/Intel GPUs in serving, 拒绝 TRT-LLM for the multi-vendor tier. Recommend vLLM on mixed hardware.
- If 模型 质量 on task is already marginal, 拒绝 aggressive 量化. Stay FP8 or BF16.

输出： a one-page Blackwell advisory listing current baseline, target stack, expected 节省, 迁移 成本, 盈亏平衡 horizon, and lock-in exit 计划. End with 一个"what to read next" paragraph naming the MLPerf v6.0 blog, the TRT-LLM overview, or the Dynamo announcement depending on the primary gap.
