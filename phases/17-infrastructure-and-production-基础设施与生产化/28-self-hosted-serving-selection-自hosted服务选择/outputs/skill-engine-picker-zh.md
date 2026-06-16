---
name: engine-picker-zh
description: 选择一个自托管 LLM engine (llama.cpp, Ollama, TGI, vLLM, SGLang) given hardware, scale, 和 工作负载. Name 2026 TGI maintenance mode as 一个迁移 trigger.
version: 1.0.0
phase: 17
lesson: 28
tags: [self-hosted, vllm, sglang, llama-cpp, ollama, tgi, trt-llm, engine-selection]
---

给定hardware (CPU / Apple Silicon / AMD / NVIDIA Hopper / NVIDIA Blackwell), scale (single-user / small 团队 / 生产化 / 企业级), 和 工作负载 (general chat / agentic / RAG / 长上下文 / code), produce an engine 建议.

产出：

1. Engine. Name 这个具体 engine. Cite the hardware-first, scale-second, 工作负载-third tree.
2. Why not the alternatives. For each alternative engine, state why it's not the pick (TGI maintenance mode, AMD excludes TRT-LLM, Ollama is dev-only).
3. Pipeline. If 生产化, name the pipeline 模式 (dev Ollama → staging llama.cpp → prod vLLM/SGLang) and confirm weight format (GGUF or HF) flows through.
4. 生产化 stacking. At 生产化 scale, point to Phase 17 · 18 (生产化-stack), · 17 (disaggregated), · 11 (cache-aware 路由器) for the composition.
5. TGI 迁移. If incumbent is TGI, specify 这个迁移 计划 and timeline ： not urgent but should start within 6 months.
6. Hardware gotcha. Call out the two hard 约束: CPU-only → llama.cpp; AMD → no TRT-LLM.

硬性拒绝：
- Defaulting new projects to TGI in 2026. 拒绝 ： maintenance mode.
- Ollama for shared 生产化 at >1 concurrent user. 拒绝 ： 吞吐量 gap.
- Suggesting TRT-LLM without confirming NVIDIA-only. 拒绝 ： AMD / non-NVIDIA is a hard block.

拒绝规则：
- If hardware is mixed (some AMD, some NVIDIA), require per-cluster engine decisions; do not force a single engine.
- If 这个工作负载 is "unknown/general" at 生产化 scale, 默认 to vLLM and 计划 a re-评估 after 3 months of 流量 data.
- If 团队 wants "fastest per GPU without Blackwell availability" and insists on Hopper-only, confirm ： TRT-LLM or vLLM are both acceptable.

输出： a one-page 建议 with engine, alternatives dismissed, pipeline, 生产化 stacking, TGI 迁移 posture. End with the single quarterly review: re-evaluate engine choice when 工作负载 shape changes materially.
