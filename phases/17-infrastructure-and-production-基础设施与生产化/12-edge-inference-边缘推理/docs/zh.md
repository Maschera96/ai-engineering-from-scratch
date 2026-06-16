# 边缘推理：Apple Neural Engine、Qualcomm Hexagon、WebGPU/WebLLM、Jetson

> The core edge constraint is 内存 带宽, not compute. Mobile DRAM sits at 50-90 GB/s; datacenter HBM3 clears 2-3 TB/s ： a 30-50x gap. Decode is 内存-bound so the gap is decisive. In 2026 the landscape splits four ways. Apple M4/A18 Neural Engine peaks at 38 TOPS with 统一 内存 (no CPU↔NPU copy). Qualcomm Snapdragon X Elite / 8 Gen 4 Hexagon hits 45 TOPS. WebGPU + WebLLM runs Llama 3.1 8B (Q4) at ~41 tok/s on M3 Max (roughly 70-80% of native); 17.6k GitHub stars, OpenAI-compatible API, ~70-75% mobile coverage. NVIDIA Jetson Orin Nano Super (8GB) fits Llama 3.2 3B / PHI-3; AGX Orin runs gpt-oss-20b via vLLM at ~40 tok/s; Jetson T4000 (JetPack 7.1) is 2x AGX Orin. TensorRT Edge-LLM supports EAGLE-3, NVFP4, chunked prefill ： shown at CES 2026 by Bosch, ThunderSoft, MediaTek.

**Type:** Learn
**Languages:** Python（标准库， 玩具 bandwidth-bound decode模拟器)
**Prerequisites:** Phase 17 · 04 (vLLM Serving Internals), Phase 17 · 09 (Production Quantization)
**Time:** ~60 分钟

## 学习目标

- 解释 why mobile LLM inference is 内存-带宽-bound and compute is secondary.
- Enumerate the four edge targets (Apple ANE, Qualcomm Hexagon, WebGPU/WebLLM, NVIDIA Jetson) and match each to a use case.
- Name the 2026 WebGPU coverage gap (Firefox Android catching up) and the Safari iOS 26 landing.
- Pick 一个量化 format per target (Core ML INT4 + FP16 for ANE, QNN INT8/INT4 for Hexagon, WebGPU Q4 for browser, NVFP4 for Jetson Thor).

## 问题

一个客户 wants an on-device chatbot: voice-first, private-by-默认, works offline. On a MacBook Pro M3 Max, Llama 3.1 8B Q4 runs at ~55 tok/s ： fine. On an iPhone 16 Pro, the same 模型 runs at 3 tok/s ： not fine. On a mid-range Android with Snapdragon 8 Gen 3, 7 tok/s. In the browser via WebGPU on Chrome Android v121+, 4-8 tok/s depending on the device.

这个吞吐量 方差 is not a porting issue. It is 这个带宽 gap times 这个量化 format times whether the NPU is accessible from user-space. 边缘推理 in 2026 is four different problems with four different solutions.

## 概念

### 带宽才是真正上限

Decode reads the full set of weights for every 词元. One 7B 模型 in Q4 is 3.5 GB. Reading 3.5 GB at 50 GB/s takes 70 ms ： a theoretical ceiling of ~14 tok/s. At 90 GB/s (high-end mobile DRAM) the ceiling moves to ~25 tok/s. No amount of compute helps below this number.

Datacenter HBM3 at 3 TB/s clears the same 3.5 GB in 1.2 ms ： ceiling is 830 tok/s. Same 模型, same weights. 不同内存 subsystem.

### Apple Neural Engine (M4 / A18)

- Up to 38 TOPS. 统一 内存 (CPU and ANE share the same pool) ： no copy overhead.
- Access via Core ML + `.mlmodel` compiled 模型, or via Metal Performance Shaders (MPS) through PyTorch.
- Llama.cpp Metal backend uses MPS, not ANE 直接; native ANE 需要 Core ML conversion.
- Best practical path for iOS apps in 2026: Core ML with INT4 weights + FP16 activations.

### Qualcomm Hexagon (Snapdragon X Elite / 8 Gen 4)

- Up to 45 TOPS. Integrated with CPU and GPU in the SoC but separate 内存 domain.
- QNN (Qualcomm Neural Network) SDK and AI Hub provide conversion from PyTorch/ONNX.
- Chat templates, Llama 3.2, PHI-3 all ship as first-class artifacts on AI Hub.

### Intel / AMD NPUs (Lunar Lake, Ryzen AI 300)

- 40-50 TOPS. Software lags behind Apple/Qualcomm; OpenVINO is improving but niche.
- Best for Windows ARM copilot apps; native on AMD/Intel desktops for local-first.

### WebGPU + WebLLM

- Run 模型 in the browser via WebGPU compute shaders; no install.
- Llama 3.1 8B Q4 at ~41 tok/s on M3 Max ： roughly 70-80% of native via same backend.
- 17.6k GitHub stars on WebLLM; OpenAI-compatible JS API; Apache 2.0.
- 2026 coverage: Chrome Android v121+, Safari iOS 26 GA, Firefox Android still catching up. Overall ~70-75% mobile coverage.

### NVIDIA Jetson family

- Orin Nano Super (8GB): fits Llama 3.2 3B, PHI-3 at good tok/s.
- AGX Orin: runs gpt-oss-20b via vLLM at ~40 tok/s.
- Thor / T4000 (JetPack 7.1): 2x AGX Orin performance, EAGLE-3 and NVFP4 supported.
- TensorRT Edge-LLM (2026) supports EAGLE-3 speculative decoding, NVFP4 weights, chunked prefill ： the datacenter optimizations ported to edge.

### 按目标选择量化

| Target | Format | Notes |
|--------|--------|-------|
| Apple ANE | INT4 weights + FP16 activations | Core ML conversion path |
| Qualcomm Hexagon | QNN INT8 / INT4 | AI Hub converters |
| WebGPU / WebLLM | Q4 MLC (q4f16_1) | Use `mlc_llm convert_weight` + compiled `.wasm`; GGUF is not supported |
| Jetson Orin Nano | Q4 GGUF or TRT-LLM INT4 | 内存-bound |
| Jetson AGX / Thor | NVFP4 + FP8 KV | Edge-LLM path |

### 边缘上的长上下文陷阱

Llama 3.1's 128K context is a datacenter 功能. On a phone with 8 GB RAM, 4 GB 模型 + 2 GB KV 缓存 for 32K 词元 + OS overhead = OOM. Edge deployments keep context at 4K-8K unless aggressive KV 量化 (Q4 KV) is accepted.

### 语音是杀手级应用

Voice agents are 延迟-sensitive (first 词元 < 500 ms). Local inference eliminates network 延迟 entirely. Combine with speech-to-text (Whisper Turbo variants run on edge) 和 边缘推理 becomes 这个生产化-质量 voice loop.

### 你应该记住的数字

- Apple M4 / A18 ANE: 38 TOPS.
- Qualcomm Hexagon SD X Elite: 45 TOPS.
- WebLLM M3 Max: ~41 tok/s on Llama 3.1 8B Q4.
- AGX Orin: ~40 tok/s on gpt-oss-20b via vLLM.
- Datacenter-edge 带宽 gap: 30-50x.
- WebGPU mobile coverage: ~70-75% (Firefox Android lagging).

## 使用它

`code/main.py` computes theoretical decode 吞吐量 ceilings from 带宽-bound math across edge targets. Compares to observed 基准 and highlights where 带宽, not compute, is the bottleneck.

## 交付它

This lesson 产出 `outputs/skill-edge-target-picker.md`. 给定平台 (iOS/Android/browser/Jetson), 模型, 和 延迟/内存 budget, picks 一个量化 format and conversion pipeline.

## 练习

1. Run `code/main.py`. For a 7B 模型 in Q4 on a Snapdragon 8 Gen 3 (~77 GB/s 带宽), compute the decode ceiling. 比较 to observed 6-8 tok/s ： is the runtime efficient?
2. WebGPU on Android 需要 Chrome v121+. Design a fallback for older browsers ： server-side via the same OpenAI-compatible API.
3. Your iOS app needs 4K-context streaming. Which 模型/format combination lets you stay under 4 GB active 内存 on an iPhone 16?
4. Jetson AGX Orin runs gpt-oss-20b at 40 tok/s. Jetson Nano fits only a 3B. If your 产品 targets both, how do you unify the inference stack?
5. Argue whether "WebLLM is 生产化-ready in 2026." Cite the coverage, performance, and the Firefox Android gap.

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| ANE | "Apple neural engine" | On-device NPU in M-series and A-series; 统一 内存 |
| Hexagon | "Qualcomm NPU" | Snapdragon NPU; QNN SDK for access |
| WebGPU | "browser GPU" | W3C-standardized browser GPU API; Chrome/Safari 2026 |
| WebLLM | "browser LLM runtime" | MLC-LLM project; Apache 2.0; OpenAI-compatible JS |
| Jetson | "NVIDIA edge" | Orin Nano / AGX / Thor / T4000 family |
| TRT Edge-LLM | "edge TensorRT" | 2026 edge port of TensorRT-LLM; EAGLE-3 + NVFP4 |
| 统一 内存 | "shared pool" | CPU and NPU see same RAM; no copy overhead |
| 带宽-bound | "内存 limited" | Decode gated by bytes/sec reading weights |
| Core ML | "Apple conversion" | Apple framework for ANE-原生模型 |
| QNN | "Qualcomm stack" | Qualcomm Neural Network SDK |

## 延伸阅读

- [On-Device LLMs State of the Union 2026](https://v-chandra.github.io/on-device-llms/) ： landscape and 基准.
- [NVIDIA Jetson Edge AI](https://developer.nvidia.com/blog/getting-started-with-edge-ai-on-nvidia-jetson-llms-vlms-and-foundation-models-for-robotics/) ： Orin / AGX / Thor.
- [NVIDIA TensorRT Edge-LLM](https://developer.nvidia.com/blog/accelerating-llm-and-vlm-inference-for-automotive-and-robotics-with-nvidia-tensorrt-edge-llm/) ： 2026 edge port announcement.
- [WebLLM (arXiv:2412.15803)](https://arxiv.org/html/2412.15803v2) ： design and 基准.
- [Apple Core ML](https://developer.apple.com/documentation/coreml) ： ANE-native conversion.
- [Qualcomm AI Hub](https://aihub.qualcomm.com/) ： pre-converted 模型 for Hexagon.
