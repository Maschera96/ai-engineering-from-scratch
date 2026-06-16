---
name: voice-cloner-zh
description: 为语音克隆部署选择克隆方式（zero-shot / conversion / adaptation）、同意工件、水印和安全过滤。
version: 1.0.0
phase: 6
lesson: 08
tags: [voice-cloning, voice-conversion, watermark, consent, safety]
---

给定任务（语言、可用参考长度、适配预算、许可证约束、同意状态、部署规模），输出：

1. 方式。Zero-shot clone（F5-TTS / VibeVoice / Orpheus / OpenVoice V2）· voice conversion（kNN-VC / OpenVoice V2 tone-color）· speaker adaptation（XTTS v2 + LoRA / VITS full fine-tune）。
2. 参考准备。所需长度、SNR（≥ 20 dB）、mono 16 kHz+、silence trim、`ref_text`（F5-TTS 必须完全匹配）。拒绝带背景音乐的参考。
3. 同意工件。声音所有者的明确录音同意。模板：姓名 + 日期 + 目的 + 范围 + 撤销流程。保存 7 年以上。
4. 水印。每个输出嵌入 AudioSeal 16-bit payload。在 CI 中配置 detector，发布音频前验证存在。
5. 安全过滤。命名实体（名人 / 政治人物 / 未成年人）prompt 拒绝；按用户每小时限流；记录每次克隆生成的审计日志；kill-switch。

拒绝没有水印策略的克隆发布。拒绝克隆命名名人、政治人物、未成年人，不管同意声明如何。拒绝短于 3 s 或 SNR < 20 dB 的参考。拒绝将 F5-TTS 用于商业部署（CC-BY-NC）。拒绝跨语言克隆却不明确标记口音迁移缺口。

示例输入："Accessibility app: let ALS patient bank their voice while still speaking, then speak through TTS after voice loss. English, US."

示例输出：
- 方式： OpenVoice V2 (MIT, zero-shot, 6 s reference). Accessibility use case with inherent consent; patient is voice owner.
- Reference prep: record 5 × 6 s clips in studio-quality conditions (quiet room, USB mic, 24 kHz). Store raw + transcripts. Build centroid reference for stability.
- Consent: digital signature + video affirmation attesting to the purpose ("post-diagnosis voice reuse"), stored on encrypted volume with 10-year retention. Revocation hotline.
- Watermark: AudioSeal 16-bit payload encoding `patient_id` + `clip_id`; detector runs on every generation in CI.
- 安全： hard-filter named-entity prompts; log every generation; ROI-limited to patient's logged-in app instance. No API exposure.
