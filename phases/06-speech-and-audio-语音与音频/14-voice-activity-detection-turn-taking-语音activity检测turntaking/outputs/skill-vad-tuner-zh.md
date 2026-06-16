---
name: vad-tuner-zh
description: 为语音智能体选择 VAD 模型、阈值、silence hangover、pre-roll 和 turn-detection 策略。
version: 1.0.0
phase: 6
lesson: 14
tags: [vad, silero, cobra, turn-detection, flush-trick]
---

给定workload（consumer / call-center / edge / accessibility；噪声 profile；语言混合；延迟），输出：

1. VAD。Silero VAD（默认）· Cobra（商业准确率）· pyannote segmentation（diarization-grade）· WebRTC VAD（legacy / tiny）。给出一句理由。
2. 参数。Threshold（0.3-0.5）、min speech（200-300 ms）、silence hangover（400-800 ms）、pre-roll（250-500 ms）。
3. 语义 turn detection。启用（LiveKit turn-detector 或 custom MLP）或不启用。理由绑定预期用户说话模式。
4. Flush trick。启用（如果 STT 支持，如 Kyutai / Deepgram）或不启用。说明预期节省延迟。
5. Guards。拒绝短于 min duration 的 speech；始终保留 pre-roll；限制 per-user silence-hangover override；VAD 服务故障时 fail-open（把所有内容当作语音）。

拒绝生产中只用 energy-only VAD，因为太容易受噪声影响。拒绝 zero silence-hangover，因为会打断用户。拒绝在有专用 Silero 可用时使用 Whisper-based VAD，因为更慢且准确率更低。

示例输入："Call-center IVR for airline rebooking. Noisy background (airport). English + Spanish. &lt; 500 ms turn detection."

示例输出：
- VAD: Cobra (commercial) for the noise-resistance advantage. Fall-back to Silero if cost prohibitive.
- Parameters: threshold 0.4 (airport noise floor is high); min speech 300 ms; silence hangover 600 ms (users often pause during IVR to read flight numbers); pre-roll 400 ms.
- Semantic turn: LiveKit turn-detector enabled — mid-sentence pauses common ("I need to change my flight... to tomorrow").
- Flush trick: enabled on Deepgram streaming. Expected savings: 400 ms → 150 ms turn-end latency.
- Guards: fail-open if Cobra/Deepgram unreachable; audit log every VAD-fire event for tuning.
