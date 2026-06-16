---
name: realtime-voice-pipeline-zh
description: 为目标端到端延迟选择传输、VAD、流式 STT、LLM、流式 TTS 和编排。
version: 1.0.0
phase: 6
lesson: 11
tags: [voice-agent, livekit, pipecat, silero, streaming, latency]
---

给定目标（latency P50/P95、语言、通道、离线 vs 云、通话量），输出：

1. 传输。WebRTC（LiveKit / Daily）· WebSocket · SIP trunking（Twilio / Telnyx）。理由绑定 jitter tolerance 和用例。
2. VAD + turn-taking。Silero VAD（open，99.5% TPR）· Cobra（commercial）· LiveKit turn-detector。给出 threshold、min speech duration、silence hang-over。
3. 流式 STT。Parakeet TDT（最快开放）· Kyutai STT（with flush trick）· Deepgram Nova-3（API，~150 ms）· Whisper-streaming。说明理由。
4. LLM + streaming。在 TTS 启动前 pin 前 20 个 token。给出模型、streaming config 和 prompt injection guardrails。
5. 流式 TTS。Kokoro-82M（~100 ms TTFA）· Orpheus · Cartesia Sonic · ElevenLabs Turbo。Voice-pack 或 cloning guard（Lesson 8）。
6. 编排。LiveKit Agents · Pipecat · Vapi · Retell · custom Rust。理由绑定团队技能和规模。
7. 观测。P50/P95/P99 per-stage histograms；false-positive interruption rate；drop-call rate；通话样本 WER。

拒绝部署会在 STT 前缓存整句的方案。拒绝不流式的 TTS。拒绝用平均延迟评估，必须要求 P95。拒绝在 > 100k minutes/month 场景下选择 managed platforms（Vapi / Retell）却不做自建成本比较。

示例输入："Voice agent for car insurance quoting. &lt; 500 ms P95. English, US. 50k minutes/week. Compliance: HIPAA-adjacent (no PII in logs)."

示例输出：
- Transport: LiveKit Agents + Twilio SIP. Proven at call-center scale, HIPAA-mode opt-in.
- VAD: Silero VAD @ threshold 0.45, min speech 220 ms, silence hang-over 400 ms. LiveKit turn-detector overlay.
- STT: Deepgram Nova-3 English (~150 ms P95); fall-back to Parakeet-TDT if on-prem audit required.
- LLM: GPT-4o streaming via OpenAI realtime API; guard against prompt injection with a post-filter; pin first 20 tokens to TTS.
- TTS: Cartesia Sonic 2 (~150 ms TTFA, voice cloning not used — predefined voice).
- Orchestration: LiveKit Agents. Observability via Hamming AI for production.
- Logs: strip CVV / SSN / DOB with a regex + NER pass before persistence. Retain 30 days.
