---
name: edge-target-picker-zh
description: 选择一个边缘推理 target (Apple ANE, Qualcomm Hexagon, WebGPU/WebLLM, NVIDIA Jetson) and matching 量化 format given device, 模型, 和 延迟 budget.
version: 1.0.0
phase: 17
lesson: 12
tags: [edge, ane, hexagon, webgpu, webllm, jetson, core-ml, qnn, nvfp4]
---

给定部署 平台 (iOS, Android, browser, robotics/automotive/edge server), 模型, 和 延迟/内存 budget, produce an edge target 建议.

产出：

1. Target. Name 这个具体 NPU/GPU (ANE, Hexagon, WebGPU, Jetson Orin Nano / AGX / Thor). 说明理由 with 这个平台 and the 2026 runtime coverage.
2. 带宽 ceiling. Compute theoretical decode ceiling: bandwidth_GB_s / model_size_GB. 比较 to the user's tok/s requirement. If the ceiling is below the requirement, 拒绝 or propose a smaller 模型 / tighter 量化.
3. 量化 format. Pick Q4 GGUF (browser/edge CPU), Core ML INT4 + FP16 (ANE), QNN INT8/INT4 (Hexagon), or NVFP4 + FP8 KV (Jetson Thor / Edge-LLM).
4. Conversion pipeline. Name the exact converter (Core ML converter, Qualcomm AI Hub, MLC-LLM for WebLLM, TensorRT-LLM Edge compiler).
5. Context budget. State the max context that fits alongside weights in device RAM. For 长上下文 use cases, specify KV 量化 (Q4 KV) 或 拒绝.
6. Fallback. When the device is incapable or WebGPU is unavailable (Firefox Android, older browsers), specify the server-side API fallback with the same OpenAI-compatible interface.

硬性拒绝：
- Promising tok/s above 带宽 ceiling. 拒绝 ： physics.
- Targeting ANE 直接 via a non-Core ML runtime in 2026. Only Core ML exposes ANE natively.
- Assuming WebGPU is on every browser. 2026 coverage is ~70-75% mobile; always specify the fallback.

拒绝规则：
- If 这个模型 is >6 GB and the target is a phone (4-8 GB RAM), 拒绝 ： propose a smaller 模型 or aggressive 量化 first.
- If 这个请求 is 128K context on a 7B 模型 on iPhone, 拒绝 ： device RAM cannot fit without Q4 KV plus sliding-window attention.
- If 这个部署 需要 长上下文 streaming on Android via WebGPU and the user 需要 Firefox support, 拒绝 and require Chrome or a server fallback.

输出： a one-page 计划 naming target, ceiling, 量化, converter, context budget, fallback. End with a single 指标: observed tok/s on the worst-case device in the target fleet.
