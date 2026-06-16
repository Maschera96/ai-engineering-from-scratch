---
name: music-designer-zh
description: 为部署选择音乐生成模型、许可证策略、长度计划和披露元数据。
version: 1.0.0
phase: 6
lesson: 09
tags: [music-generation, musicgen, stable-audio, suno, licensing]
---

给定brief（instrumental vs song、长度、商业 vs 研究、genre、预算），输出：

1. 模型。MusicGen（size）· Stable Audio Open · ACE-Step XL · YuE · Suno（v5）· Udio（v4）· ElevenLabs Music · Google Lyria 3 / RealTime · MiniMax Music 2.5，并给出一句理由。
2. 许可证和权利。生成片段的商业许可、署名（CC）、非商业限制、owned catalog fine-tune。记录 rightsholder 和 chain。
3. 长度与结构。单次生成、chunked + crossfade、bridge inpainting、需要编辑时 stem separation。明确处理 30 秒 drift wall。
4. Prompt schema。Key / BPM / genre / instrumentation，加上声乐模型的 lyrics 和 mood tags。限制名人姓名和商标化风格标签。
5. 披露与元数据。Watermark（适用时 AudioSeal）、`isAIGenerated` 元数据 tag、EU AI Act / CA SB 942 的 AI-disclosure overlay。

拒绝开放模型上的 celebrity-style prompts。拒绝把非商业许可证生成物（Stable Audio Open）用于付费产品。拒绝没有披露标签的声乐音乐部署。标记依赖 Udio stems 的 stem-editing pipeline，那些受商业条款约束，不是免费使用。

示例输入："Background music for a meditation app. Instrumental. Full commercial rights required. Up to 5 min per track."

示例输出：
- 模型： MusicGen-large (MIT) for instrumental with full commercial rights. No Stable Audio (non-commercial).
- 许可证： MIT — commercial rights retained by deployer. Track rightsholder: app company.
- 长度： chunk into 30s segments with 3s crossfade; 10 generations concatenated → 5 min. Add a subtle ambient fade-in/out envelope to hide drift.
- Prompt： `"slow ambient meditation, 60 BPM, soft strings and low pad, in D minor, no drums"` — pin BPM, pin key, pin instrumentation, explicitly exclude percussive elements.
- 披露： `"AI-generated music"` tag in app credits; metadata `creator=AI-Gen:MusicGen-large, date=<iso>`. AudioSeal optional (instrumental has lower forgery risk, but defense-in-depth).
