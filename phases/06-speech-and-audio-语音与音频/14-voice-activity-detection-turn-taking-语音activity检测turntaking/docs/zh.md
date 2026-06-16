# 语音活动检测与轮次判断，Silero、Cobra 和 Flush 技巧

> 每个语音智能体都由两个决定决定生死：用户现在是否在说话，用户是否说完了。VAD 回答第一个问题。轮次检测，也就是 VAD 加静音 hangover 加语义 endpoint 模型，回答第二个问题。任一出错，助手就会打断用户或永远不闭嘴。

**Type:** Build
**Languages:** Python
**Prerequisites:** Phase 6 · 11（实时音频），Phase 6 · 12（语音助手）
**Time:** ~45 分钟

## 问题

每个 20 ms chunk 都要做三件事：这一帧是否是语音，用户是否开始了新 utterance，用户是否已经结束这一轮。

朴素能量阈值会被交通、键盘、背景人声轻易击穿。2026 默认是 Silero VAD，加语义 turn detector，再加校准过的静音 hangover。

## 概念

![VAD cascade: energy → Silero → turn-detector → flush trick](../assets/vad-turn-taking.svg)

三层 VAD cascade：能量 gate 最便宜，过滤明显静音；Silero VAD 是开放默认，CPU 上每 30 ms chunk 约 1 ms；语义 turn detector 区分句中停顿和真正说完。

关键参数包括阈值、最短语音时长、静音 hangover 和 pre-roll buffer。阈值低会少切首词，但多误报；hangover 太短会打断用户，太长会显慢。

Kyutai 的 flush trick 在 VAD 判定结束时给 STT 发送 flush 信号，强制立刻输出，减少等待 look-ahead 的延迟。

## Build It

### Step 1: 读取与检查

实现能量 gate，计算 RMS 和 dBFS。

```python
def energy_vad(chunk, threshold_dbfs=-40.0):
    rms = (sum(x * x for x in chunk) / len(chunk)) ** 0.5
    dbfs = 20.0 * math.log10(max(rms, 1e-10))
    return dbfs > threshold_dbfs
```

### Step 2: 构建核心表示

用 Silero VAD 得到 speech timestamps，并设置 threshold、min_speech_duration_ms、min_silence_duration_ms 和 speech_pad_ms。

```python
from silero_vad import load_silero_vad, get_speech_timestamps

vad = load_silero_vad()
audio = torch.tensor(waveform_16k, dtype=torch.float32)
segments = get_speech_timestamps(
    audio, vad, sampling_rate=16000,
    threshold=0.5,
    min_speech_duration_ms=250,
    min_silence_duration_ms=500,
    speech_pad_ms=300,
)
for s in segments:
    print(f"{s['start']/16000:.2f}s - {s['end']/16000:.2f}s")
```

### Step 3: 运行 baseline

实现 turn-end 状态机，处理 pre-roll、speech、hangover 和 end event。

```python
class TurnDetector:
    def __init__(self, silence_hangover_ms=500, min_speech_ms=250):
        self.state = "idle"
        self.speech_ms = 0
        self.silence_ms = 0
        self.silence_hangover_ms = silence_hangover_ms
        self.min_speech_ms = min_speech_ms

    def update(self, is_speech, chunk_ms=20):
        if is_speech:
            self.speech_ms += chunk_ms
            self.silence_ms = 0
            if self.state == "idle" and self.speech_ms >= self.min_speech_ms:
                self.state = "speaking"
                return "START"
        else:
            self.silence_ms += chunk_ms
            if self.state == "speaking" and self.silence_ms >= self.silence_hangover_ms:
                self.state = "idle"
                self.speech_ms = 0
                return "END"
        return None
```

### Step 4: 升级生产方案

写 flush trick 的骨架，在 turn end 时通知支持 flush 的 STT。

```python
def flush_on_end(stt_client, audio_buffer):
    stt_client.send_audio(audio_buffer)
    stt_client.send_flush()
    return stt_client.recv_transcript(timeout_ms=150)
```

## Use It

Silero 是默认选择，Cobra 是合规或高噪声准确率升级，WebRTC VAD 是 legacy tiny 选项。生产不要只用能量阈值。每个部署都要用真实噪声样本调 threshold 和 hangover。

## Ship It

保存为 `outputs/skill-vad-tuner.md`。这个 skill 帮你为语音智能体选择 VAD 模型、阈值、静音 hangover、pre-roll 和轮次检测策略。

## Exercises

1. 在安静、键盘声和街道噪声下比较能量 VAD 和 Silero。
2. 把 hangover 从 300 ms 调到 900 ms，记录用户感知延迟。
3. 实现 pre-roll，确认“hey”不会被切掉。

## Key Terms

| 术语 | 常见说法 | 实际含义 |
|------|----------|----------|
| VAD | 语音活动检测 | 判断当前帧是否有人声。 |
| hangover | 结束前等待时间 | 避免句中停顿被当作结束。 |
| pre-roll | 触发前缓存 | 避免切掉开头。 |
| turn-taking | 轮次判断 | 判断用户是否说完。 |
| flush trick | 流式 STT 刷新技巧 | 减少转写尾延迟。 |

## Further Reading

- [Silero VAD](https://github.com/snakers4/silero-vad)，the reference open VAD.
- [Picovoice Cobra VAD](https://picovoice.ai/products/cobra/)，commercial accuracy leader.
- [Kyutai，Unmute + flush trick](https://kyutai.org/stt)，the sub-200 ms engineering trick.
- [LiveKit，turn detection](https://docs.livekit.io/agents/logic/turns/)，semantic endpointing in production.
- [WebRTC VAD](https://webrtc.googlesource.com/src/)，the legacy baseline.
- [pyannote segmentation](https://github.com/pyannote/pyannote-audio)，diarization-grade segmentation.
