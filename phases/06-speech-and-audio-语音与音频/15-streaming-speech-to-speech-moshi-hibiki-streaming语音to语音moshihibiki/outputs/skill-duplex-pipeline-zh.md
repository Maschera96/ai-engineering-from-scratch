---
name: duplex-pipeline-zh
description: 为语音智能体 workload 选择 full-duplex（Moshi）还是 pipeline（VAD + STT + LLM + TTS）架构。
version: 1.0.0
phase: 6
lesson: 15
tags: [moshi, hibiki, full-duplex, voice-agent, streaming]
---

给定workload（延迟目标、工具调用需求、语言覆盖、硬件预算、cloud vs edge），输出：

1. 架构。Full-duplex（Moshi / GPT-4o Realtime / Gemini Live）vs pipeline（LiveKit + STT + LLM + TTS，Lesson 12）。给出一句理由。
2. 模型。Moshi · Hibiki · Hibiki-Zero · Sesame CSM · GPT-4o Realtime · Gemini 2.5 Live · traditional pipeline。说明理由。
3. 规模。Per-session GPU cost（Moshi 占一个 slot）、最大并发 sessions、cold-start impact。
4. 工具调用路径。如果需要，选择 hybrid pipeline（duplex + external LLM for tool calls）或 pure pipeline。解释取舍。
5. 语言覆盖。Full-duplex 模型语言支持较窄；pipeline 继承 LLM 的多语言能力。

拒绝给需要工具调用或检索的企业智能体使用 full-duplex-only 架构，Moshi 是对话模型，不是智能体框架。拒绝给 sub-250 ms 对话智能体使用 pipeline-only，因为阶段延迟会叠加。拒绝在单 GPU 上让 Moshi 承载 > 4 concurrent sessions，因为会打到 contention。

示例输入："Voice companion for language learning — conversational fluency practice. English + French. &lt; 250 ms responsiveness. 10k daily actives."

示例输出：
- 架构： full-duplex (Moshi). Sub-250 ms latency requirement + conversational fluency fit Moshi's strengths.
- 模型： Moshi. EN + FR both well-supported. CC-BY 4.0 license.
- 规模： one L4 GPU per 4-6 concurrent sessions → ~1500 GPUs at peak for 10k DAU at 10% concurrency. Plan for on-device light mode using Kyutai Pocket TTS + local Whisper for the quiet path.
- Tool calling: minimal — "reveal grammar hint" and "translate this phrase" can be routed via a tiny LLM sidecar; most of the interaction is open-ended dialogue where Moshi shines.
- Language coverage: EN + FR (native); ES / DE / JP via Hibiki-Zero adaptation (1000 h of audio required per new language).
