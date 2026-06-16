---
name: alm-picker-zh
description: 为音频理解任务选择音频语言模型、benchmark 子集、输出模态（文本 vs 语音）和 guardrails。
version: 1.0.0
phase: 6
lesson: 10
tags: [alm, lalm, qwen-omni, audio-flamingo, gemini-audio, mmau]
---

给定任务（speech / sound / music / multi-audio / long-audio、输出模态、延迟、许可证），输出：

1. 模型。Qwen2.5-Omni-7B · Qwen3-Omni · SALMONN · Audio Flamingo 3 · AF-Next · LTU · GAMA · Gemini 2.5 Pro（API）· GPT-4o Audio（API），并给出一句理由。
2. 要验证的 benchmark 子集。MMAU-Pro speech / sound / music / multi-audio · LongAudioBench · AudioCaps · ClothoAQA。选择匹配用户任务的轴。
3. 输出模态。Text-only · text + speech（Qwen-Omni、GPT-4o Audio）。需要时为额外 speech decoder 预留预算。
4. Guardrails。当模型 multi-audio 分数 < 30%（接近随机）时，拒绝需要 multi-audio comparison 的 prompt。对 > 10 分钟输入，在 LALM 前先 diarize。
5. 升级路径。任务何时回退到专用模型，如 Whisper 转写、BEATs 分类、pyannote diarization。LALM 不是每个细分任务的最优解。

拒绝在未验证模型 MMAU-Pro multi-audio 子集得分 > 40% 时交付 multi-audio comparison 任务。拒绝没有 upstream diarization 的长音频（> 10 min）。标记只使用 vendor-reported numbers 且未独立复验的部署。

示例输入："Compliance audit: transcribe 10-minute bank-call recordings + detect if the agent read the mandatory disclosure."

示例输出：
- 模型： Whisper-large-v3-turbo for transcription + Gemini 2.5 Pro (via API) for disclosure-check QA over the transcript. LALM direct on raw audio is tempting but long-audio LALM accuracy drops past 10 min.
- Benchmark subset: MMAU-Pro speech subset (Gemini 2.5 Pro = 73.4%) — covers the speech-reasoning axis. Also spot-check on your own 50-call gold set.
- Output modality: text-only. Speech output not needed for an audit report.
- Guardrails: diarize with pyannote 3.1 first; send per-speaker segments separately; log confidence score per call.
- Escalation: if a call fails the disclosure check, route to human reviewer instead of autonomous flag.
