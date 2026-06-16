# 音频基础，波形、采样、傅里叶变换

> 波形是原始信号。频谱图是表示方式。Mel 特征是更适合机器学习的形式。现代 ASR 和 TTS 流水线都会沿着这条阶梯前进，第一步就是理解采样和傅里叶。

**Type:** Learn
**Languages:** Python
**Prerequisites:** Phase 1 · 06（向量与矩阵），Phase 1 · 14（概率分布）
**Time:** ~45 分钟

## 问题

麦克风产生的是随时间变化的声压信号，神经网络消费的是张量。两者之间有一整套约定，采样率、通道、量化、频率表示，只要其中一个错了，就会出现安静但昂贵的错误：训练看起来正常，WER 却翻倍，TTS 输出带嘶声，或者语音克隆系统记住的是麦克风而不是说话人。

语音系统里的很多问题都可以追到三个问题：数据用什么采样率录制，模型期望什么采样率；信号是否发生混叠；你处理的是原始采样还是频率表示。把这些答对，Phase 6 后面的内容就可控。答错，即使 Whisper-Large-v4 也会输出垃圾。

## 概念

![波形, 采样, DFT, and frequency bins visualized](../assets/audio-fundamentals.svg)

波形是一维浮点数组，通常落在 `[-1.0, 1.0]`。用样本编号索引。把样本编号换成秒，需要除以采样率：`t = n / sr`。16 kHz 的 10 秒片段有 160,000 个浮点数。

采样率 `sr` 表示每秒采多少个样本。电话常见 8 kHz，ASR 标准是 16 kHz，现代 TTS 常用 24 kHz，音乐和专业音频常用 44.1 kHz 或 48 kHz。

Nyquist-Shannon 定理说，采样率 `sr` 只能无歧义表示最高 `sr/2` 的频率。高于 Nyquist 的能量会折叠到低频，污染信号。因此降采样前必须先低通滤波。

位深决定每个样本的分辨率。16-bit PCM 是最通用的交换格式，音乐常用 24-bit，DSP 内部常用 32-bit float。

傅里叶变换把有限信号写成不同频率正弦波的和。DFT 对 `N` 个样本输出 `N` 个复系数，每个系数对应一个频率 bin。FFT 是 DFT 的 `O(N log N)` 快速算法。

真实音频不会一次对整段做 FFT，而是切成重叠帧，乘 Hann 或 Hamming 窗，再对每帧做 FFT，这就是 STFT。Lesson 02 从这里继续。

| 采样率 | 用途 |
|------|-----|
| 8 kHz | 电话和旧 VOIP。Nyquist 只有 4 kHz，会伤害辅音，不建议 ASR。 |
| 16 kHz | ASR 标准。Whisper、Parakeet、SeamlessM4T v2 都消费 16 kHz。 |
| 22.05 kHz | 较旧 TTS 声码器训练。 |
| 24 kHz | 现代 TTS，如 Kokoro、F5-TTS、xTTS v2。 |
| 44.1 kHz | CD 音频和音乐。 |
| 48 kHz | 影视、专业音频和高保真 TTS。 |

```figure
mel-scale
```

## Build It

### Step 1: 读取与检查

读取 WAV，画出波形，并检查采样率、通道数和浮点范围。演示代码用 stdlib 的 `wave` 保持无依赖，生产环境通常用 `soundfile` 或 `torchaudio.load`。

```python
import soundfile as sf
waveform, sr = sf.read("clip.wav", dtype="float32")  # shape (T,), sr=int
```

### Step 2: 构建核心表示

从零合成 440 Hz 正弦波，理解样本数、频率和振幅的关系，再写成 16-bit PCM。

```python
import math

def sine(freq_hz, sr, seconds, amp=0.5):
    n = int(sr * seconds)
    return [amp * math.sin(2 * math.pi * freq_hz * i / sr) for i in range(n)]
```

### Step 3: 运行 baseline

手写 DFT。`O(N²)` 只适合小 `N` 校验正确性，真实音频应使用 `numpy.fft.rfft` 或 `torch.fft.rfft`。

```python
def dft(x):
    N = len(x)
    out = []
    for k in range(N):
        re = sum(x[n] * math.cos(-2 * math.pi * k * n / N) for n in range(N))
        im = sum(x[n] * math.sin(-2 * math.pi * k * n / N) for n in range(N))
        out.append((re, im))
    return out
```

### Step 4: 升级生产方案

用幅度峰值索引 `k_star` 推回主频：`k_star * sr / N`。

### Step 5: 验证边界

采样 10 kHz 下的 7 kHz 正弦波，观察它折叠成 3 kHz，直观看到混叠。

## Use It

2026 年可交付栈：`soundfile` 读写音频，`torchaudio.transforms.Resample` 或 `librosa.resample` 重采样，`torchaudio` 或 `librosa` 提取 STFT 和 Mel，实时流用 `sounddevice` 或 `pyaudio`，文件探查用 `ffprobe` 或 `soxi`。决策规则：先匹配采样率，再做任何别的事。

## Ship It

保存为 `outputs/skill-audio-loader.md`。这个 skill 帮你检查音频输入是否符合下游模型期望，并在不符合时安全重采样。

## Exercises

1. 合成 16 kHz、1 秒的 220 Hz + 440 Hz + 880 Hz 混合波，运行 DFT，确认三个峰值。
2. 录一段 48 kHz 语音，分别用带抗混叠的重采样和朴素抽取降到 16 kHz，对比 FFT 中混叠出现的位置。
3. 只用 `math` 和手写 DFT 实现 STFT，帧长 400，hop 160，Hann 窗，用 `matplotlib.pyplot.imshow` 画出幅度。

## Key Terms

| 术语 | 常见说法 | 实际含义 |
|------|----------|----------|
| 采样率 | 每秒样本数 | ADC 测量信号的频率，单位 Hz。 |
| Nyquist | 可表示最高频率 | `sr/2`，高于它的能量会混叠回低频。 |
| 位深 | 每个样本的分辨率 | `int16` 有 65,536 级，`float32` 在 `[-1, 1]` 中约有 24-bit 精度。 |
| DFT | 序列上的傅里叶变换 | `N` 个样本变成 `N` 个复频率系数。 |
| FFT | 快速 DFT | `O(N log N)` 算法，通常要求 `N` 是 2 的幂。 |
| Bin | 频率列 | `k · sr / N` Hz，分辨率是 `sr / N`。 |
| STFT | 频谱图的底层机制 | 随时间做分帧、加窗 FFT。 |
| 混叠 | 奇怪的频率幽灵 | Nyquist 以上能量镜像到低频 bin。 |

## Further Reading

- [Shannon (1949). Communication in the Presence of Noise](https://people.math.harvard.edu/~ctm/home/text/others/shannon/entropy/entropy.pdf)，the paper behind the sampling theorem.
- [Smith，The Scientist and Engineer's Guide to Digital Signal Processing](https://www.dspguide.com/ch8.htm)，free, canonical DSP textbook.
- [librosa docs，audio primer](https://librosa.org/doc/latest/tutorial.html)，practical walkthrough with code.
- [Heinrich Kuttruff，Room Acoustics (6th ed.)](https://www.routledge.com/Room-Acoustics/Kuttruff/p/book/9781482260434)，reference for why real-world audio is not a clean sinusoid.
- [Steve Eddins，FFT Interpretation notebook](https://blogs.mathworks.com/steve/2020/03/30/fft-spectrum-and-spectral-densities/)，frequency bin intuition cleared up in 10 minutes.
