---
name: spoof-defender-zh
description: 为语音生成或语音认证部署选择检测模型、水印、溯源 manifest 和运营手册。
version: 1.0.0
phase: 6
lesson: 16
tags: [anti-spoofing, watermark, audioseal, asvspoof, c2pa, voice-fraud]
---

给定workload（voice-gen vs voice-auth、部署规模、合规地区、对手画像），输出：

1. 检测（CM）。AASIST · RawNet2 · NeXt-TDNN + WavLM · commercial（Pindrop、Validsoft）。训练数据：ASVspoof 2019 / ASVspoof 5 / domain-specific。目标 EER。
2. 水印（outbound gen）。AudioSeal 16-bit payload 编码 `(model_id, user_id, generation_ts)` · WaveVerify（替代）· none（需说明理由）。Detector 在每个输出 pre-ship 前于 CI 运行。
3. 溯源。C2PA manifest，用部署方 key 签名 · IPTC metadata · none（非消费者音频）。
4. 语音认证 guards（如适用）。Liveness challenge（random phrase TTS + transcribe）、replay attack detection（AASIST + PA model）、按通道校准 biometric threshold。
5. 运营。Audit log retention、consent artifact retention（7+ years）、abuse-detection signals（sudden volume burst、named-entity prompts）、kill-switch procedure。

拒绝没有 AudioSeal（或等价水印）的 voice-gen 部署。拒绝没有 anti-spoofing detection 的语音生物识别部署，语音克隆让 cosine-only auth 很容易被绕过。拒绝只依赖 provenance manifest 的部署，因为它可被剥离。拒绝把 ASVspoof 2019 训练出的检测阈值直接用于真实部署而不做通道校准 sweep。

示例输入："Bank customer-service IVR. Voice biometric unlock + AI-generated voice agent. 10M calls/month. US + EU."

示例输出：
- 检测： Pindrop commercial (preferred) or NeXt-TDNN + WavLM open. Training on ASVspoof 5 + 100k bank-specific call samples. Target EER &lt; 0.5% on in-domain data.
- 水印： AudioSeal 16-bit payload on every outbound TTS utterance; payload encodes bank_id + session_id + timestamp. Detector verifies before transmit.
- 溯源： C2PA manifest on audio-export-to-customer workflows; internal-only calls skip.
- Voice-auth: liveness challenge at every auth (TTS random 4-digit phrase; user repeats + detector + transcriber). Anti-spoofing runs on every inbound auth attempt. Biometric threshold at FAR 0.1%, FRR 1%.
- 运营： 7-year retention on consent + audit log in region (EU data EU-resident). Alert on sudden clone-request volume &gt; 2σ; kill-switch on abuse detection.
