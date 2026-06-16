---
name: voice-assistant-architect-zh
description: 为给定 workload 产出全栈语音助手规格，包括组件、延迟预算、观测和合规。
version: 1.0.0
phase: 6
lesson: 12
tags: [voice-assistant, architecture, livekit, pipecat, compliance]
---

给定用例（consumer / customer-support / accessibility / edge）、预期规模（并发 sessions、minutes/month）、语言、延迟目标、合规（HIPAA、PCI、EU AI Act、CA SB 942），输出：

1. 组件（7 层）。Mic + chunking · VAD · streaming STT · LLM + tools · streaming TTS · playback · interruption handler。为每层命名准确 provider/model。
2. 延迟预算。每阶段 P50 / P95 / P99 目标，加总到端到端目标。标记哪些阶段独立，哪些阶段顺序。
3. 工具调用 schema。每个 tool 的 JSON spec + error handling + fallback text。始终包含一个 LLM 在失败两次后必须走的“不能帮忙”路径。
4. 安全。Prompt injection guard、voice-cloning lockout（如果 TTS 能克隆）、wake-word gate（always-on 场景）、日志 PII redact、30-day retention。
5. 观测。每阶段 P50/P95/P99、false-interruption rate、tool-call success rate、每 100 通 WER、cost per minute、abandon rate。
6. 合规。披露音频（“This is an AI assistant”）、region-pinning（EU 数据在 EU）、audit log retention、opt-out pathway。

拒绝没有 wake word 的 always-on 部署。拒绝不流式的 TTS，因为它会增加 utterance-length latency。拒绝没有 P95 的平均延迟报告，tail 才是用户流失点。拒绝在没有法律审查的情况下保留原始音频超过 30 天。

示例输入："Accessibility assistant for low-vision users: voice-only interface to a consumer email app. English. P95 &lt; 600 ms. ~10k concurrent users."

示例输出：
- 组件： sounddevice (WebRTC via LiveKit Agents) · Silero VAD · Deepgram Nova-3 (English) · GPT-4o with email tools (read_message, compose_reply, mark_read) · Cartesia Sonic 2 streaming · WebRTC out · interrupt=cancel-LLM-and-TTS on VAD fire.
- 预算： capture 120 ms + VAD 40 + STT 150 + LLM TTFT 100 + TTS TTFA 150 = 560 ms P95.
- 工具： read_message({id}), compose_reply({message_id, body}), mark_read({id}), search({query}). All return JSON; LLM has max 2 retries per tool then fallback "I couldn't do that — try rephrasing".
- 安全： prompt-injection guard (detect `ignore previous instructions`); wake word "Hey Mail"; no voice cloning (fixed Cartesia voice); redact email bodies in logs.
- Observability: Hamming AI production monitoring; per-stage Prometheus histograms; alert on false-interrupt &gt; 5% or p95 &gt; 800 ms.
- 合规： AI disclosure on first use; HIPAA opt-in for medical messages only; EU users hit EU-hosted Cartesia + GPT-4o Ireland.
