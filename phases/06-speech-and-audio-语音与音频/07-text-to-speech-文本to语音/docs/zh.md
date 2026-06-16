# 文本转语音（TTS），从 Tacotron 到 F5 和 Kokoro

> ASR 把语音反转成文本，TTS 把文本反转成语音。2026 栈由三部分组成：文本到 token，token 到 Mel，Mel 到波形。每一部分都有能在笔记本上跑的默认模型。

**Type:** Build
**Languages:** Python
**Prerequisites:** Phase 6 · 02（频谱图与 Mel），Phase 5 · 09（Seq2Seq），Phase 7 · 05（完整 Transformer）
**Time:** ~75 分钟

## 问题

TTS 失败通常不是“声音不够像人”，而是文本前端漏了数字、日期、缩写、多音字，或者声码器和 acoustic model 配置不匹配。

生产 TTS 要同时考虑自然度、可懂度、说话人相似度、延迟、授权和安全。

## 概念

![Tacotron, Fast语音, VITS, F5/Kokoro side-by-side](../assets/tts.svg)

经典 TTS 流水线是文本正则化和音素化，声学模型生成 Mel 或 codec token，声码器生成波形。

Tacotron 代表 attention 到 Mel 的旧路线，VITS 和 StyleTTS 结合生成式建模，F5-TTS 和 Kokoro 代表 2026 轻量高质量默认。

声码器从 Griffin-Lim 到 WaveNet、HiFi-GAN，再到神经 codec 解码器。

评估看 MOS/UTMOS、ASR round-trip CER/WER、克隆任务的 SECS，以及 TTFA 延迟。

## Build It

### Step 1: 读取与检查

音素化输入，处理数字、日期、缩写和 OOV。

```python
from phonemizer import phonemize
ph = phonemize("Hello world", language="en-us", backend="espeak")
# 'həloʊ wɜːld'
```

### Step 2: 构建核心表示

运行 Kokoro，作为 2026 CPU 默认 TTS。

```python
from kokoro import KPipeline
tts = KPipeline(lang_code="a")  # "a" = American English
audio, sr = tts("Please remind me to water the plants at 6 pm.", voice="af_bella")
# audio: float32 tensor, sr=24000
```

### Step 3: 运行 baseline

运行 F5-TTS 语音克隆，注意 `ref_text` 必须匹配参考音频。

```python
from f5_tts.api import F5TTS
tts = F5TTS()
wav = tts.infer(
    ref_file="my_voice_5s.wav",
    ref_text="The quick brown fox jumps over the lazy dog.",
    gen_text="Please remind me to water the plants.",
)
```

### Step 4: 升级生产方案

从零写 HiFi-GAN 声码器的概念骨架。

```python
class HiFiGAN(nn.Module):
    def __init__(self, mel_channels=80, upsample_rates=[8, 8, 2, 2]):
        super().__init__()
        # 4 upsample blocks, total 256x to go from mel-rate to audio-rate
        ...
    def forward(self, mel):
        return self.blocks(mel)  # -> waveform
```

### Step 5: 验证边界

把文本前端、声学模型和声码器串成完整流水线。

```python
text = "Please remind me at 6 pm."
phones = phonemize(text)
mel = acoustic_model(phones, speaker=alice)      # [T, 80]
wav = vocoder(mel)                                # [T * 256]
soundfile.write("out.wav", wav, 24000)
```

## Use It

英语 CPU 快速 TTS 可选 Kokoro。零样本克隆可选 F5-TTS 或 XTTS v2，但要检查许可证。多语言、低延迟或商用 API 要按语言、延迟和授权筛选。任何生产 TTS 都必须有文本 normalizer。

## Ship It

保存为 `outputs/skill-tts-designer.md`。这个 skill 帮你选择 TTS 模型、声音、文本正则化范围和评估计划。

## Exercises

1. 为含数字、日期和 URL 的文本写 normalizer。
2. 比较 Kokoro 与 F5-TTS 在 TTFA 和可懂度上的差异。
3. 用 Whisper 对 TTS 输出做 round-trip WER。

## Key Terms

| 术语 | 常见说法 | 实际含义 |
|------|----------|----------|
| TTS | 文本转语音 | 从文本生成可听语音。 |
| 音素 | 语言的最小语音单位 | TTS 前端常用中间表示。 |
| 声码器 | 从特征到波形的模型 | 如 HiFi-GAN 或 codec decoder。 |
| UTMOS | 学习式 MOS 预测器 | 快速估计自然度。 |
| TTFA | 首段音频延迟 | 流式 TTS 关键指标。 |

## Further Reading

- [Shen et al. (2017). Tacotron 2](https://arxiv.org/abs/1712.05884)，the seq2seq baseline.
- [Kim, Kong, Son (2021). VITS](https://arxiv.org/abs/2106.06103)，end-to-end flow-based.
- [Chen et al. (2024). F5-TTS](https://arxiv.org/abs/2410.06885)，current open-source SOTA.
- [Kong, Kim, Bae (2020). HiFi-GAN](https://arxiv.org/abs/2010.05646)，the vocoder that still ships in 2026.
- [Kokoro-82M on HuggingFace](https://huggingface.co/hexgrad/Kokoro-82M)，2024 CPU-friendly English TTS.
