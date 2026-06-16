---
name: voice-pipeline-zh
description: 搭建一个 Pipecat 风格的语音流水线（VAD + STT + LLM + TTS + transport），支持打断（barge-in）、置信度门控和延迟预算强制。
version: 1.0.0
phase: 14
lesson: 22
tags: [voice, pipecat, livekit, webrtc, latency]
---

给定一份语音产品规格（语言、transport、提供商），搭建一个基于帧（frame）的流水线。

产出：

1. `Frame` 类型，包含 `kind`、`payload`、`direction`（downstream / upstream）。
2. 处理器：`VAD`、`STT`、`LLM`、`TTS`、`Transport`。每个都带有 `process(frame)`。
3. `link()` 辅助函数，将处理器向前和向后链接起来。
4. 取消帧（cancel frame）处理：UPSTREAM 路径从 transport 到 TTS 到 LLM 到 STT，在每个阶段丢弃待处理的工作。
5. 观察者：每阶段的延迟指标；为每个穿过处理器的帧发出一个 OTel span（第 23 课）。
6. STT 上的置信度门控：低于阈值时，发出一个“请重复”文本帧，而不是转写结果。

硬性拒绝项：

- 没有 UPSTREAM 处理的流水线。打断（barge-in）对语音来说不是可选项。
- 没有流式输出的 LLM 调用。首字延迟占主导；必须采用流式。
- 对置信度无感知的 STT。把错误的转写喂给 LLM 会产生错误的回复。

拒绝规则：

- 如果冷启动运行时端到端延迟超过 1500ms，拒绝上线。优化整条链路或使用 MultimodalAgent（LiveKit 直连音频）。
- 如果产品以电话（telephony）为先，而流水线没有 SIP 适配器，拒绝。通过 LiveKit SIP 或某个平台（Vapi/Retell）进行路由。
- 如果产品在传输过程中承载了未加密的 PII 音频，拒绝。

输出：`frames.py`、`processors.py`、`pipeline.py`、`observers.py`、`README.md`，解释延迟预算、打断（barge-in）设计和 transport 选型。以“接下来读什么”结尾，指向第 23 课（OTel）、第 24 课（可观测性后端）或 LiveKit 文档中关于 WebRTC 细节的内容。
