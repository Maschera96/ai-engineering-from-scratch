---
name: audio-evaluator-zh
description: 为任何音频模型发布选择指标、benchmark、归一化规则和报告格式。
version: 1.0.0
phase: 6
lesson: 17
tags: [evaluation, wer, mos, utmos, eer, der, fad, mmau, leaderboard]
---

给定任务（ASR / TTS / cloning / speaker-verif / diarization / classification / music / LALM / streaming S2S），输出：

1. 主指标。WER · MOS · UTMOS · SECS · EER · DER · mAP · FAD · MMAU-Pro accuracy · latency P95。只选一个。
2. 次指标。1 到 3 个额外轴（speed、diversity、robustness）并说明原因。
3. 归一化规则。Lowercase、punctuation-strip、number expansion、whitespace collapse。使用 Whisper-normalizer 或自定义，并记录。
4. 公开 benchmark。要对比的 canonical leaderboard（Open ASR、TTS Arena、MMAU-Pro、VoxCeleb1-O、AudioSet、LongAudioBench 等）。
5. 内部集。Held-out domain data，含 N samples；demographic / acoustic slice breakdown。
6. 报告格式。Distribution（延迟 P50/P95/P99；分类 per-class recall；MMAU per-category）。Release notes template。

拒绝 latency 的单数字评估，必须报告 percentiles。拒绝 classification 只给 aggregate，必须报告 per-class。拒绝 TTS 发布缺少 MOS/UTMOS 以及克隆时缺少 SECS。拒绝 ASR 发布没有 WER normalization spec。拒绝音乐发布只给 FAD，必须配 human MOS panel。

示例输入："Release of a new English-Spanish conversational TTS. Need to convince the team it's better than the existing Cartesia-Sonic baseline."

示例输出：
- 主指标： UTMOS (paired audio samples on 50 prompts per language) + human-panel MOS (20 listeners per language, blind A/B vs baseline).
- 次指标： TTFA median & P95 (must match baseline); SECS &gt; 0.80 vs a fixed voice reference (no speaker regression); CER on round-trip ASR (Whisper-large-v3-turbo) &lt; 2%.
- 归一化： Whisper-normalizer English + Hugging Face multilingual-normalizer Spanish for round-trip WER.
- 公开 benchmark： TTS Arena (English) and Artificial Analysis Speech for relative ELO positioning. Target: within 50 ELO of the closest competitor.
- 内部集： 200 held-out prompts (100 per lang) covering money, dates, product names, 2-sentence narration, emotional read, code-switched. 10 demographic voices.
- 报告： release note with headline (UTMOS + MOS), P50/P95 TTFA histogram, SECS CDF, CER per-category breakdown, failure-mode callouts (code-switched prompts failed at X%).
