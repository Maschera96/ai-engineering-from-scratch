# 频谱图、Mel 尺度与音频特征

> 神经网络不擅长直接吃原始波形。它们更擅长吃频谱图，更擅长吃 Mel 频谱图。2026 年每个 ASR、TTS 和音频分类器，成败都取决于这个预处理选择。

**Type:** Build
**Languages:** Python
**Prerequisites:** Phase 6 · 01（音频基础）
**Time:** ~45 分钟

## 问题

原始波形很长，局部变化快，而且对相位、响度、噪声非常敏感。模型需要一种同时保留时间和频率信息的表示。

频谱图把波形切成短帧，对每帧做 FFT，得到时间乘频率的矩阵。Mel 频谱图再把线性频率压成更接近人耳感知的刻度。

## 概念

![波形 to STFT to mel spectrogram to MFCC ladder](../assets/mel-features.svg)

频谱图是 STFT 幅度随时间的堆叠。横轴是时间帧，纵轴是频率 bin，值是能量。

Mel 尺度把低频分得细，高频分得粗，因为人耳对低频差异更敏感。ASR 和 TTS 常用 80 个或 128 个 Mel bin。

Log-mel 对能量取对数，把动态范围压小。大多数现代 ASR 模型，包括 Whisper 和 Parakeet，都吃 log-mel。

MFCC 是对 log-mel 再做 DCT 得到的低维倒谱特征，适合传统模型，不适合 Whisper 这类端到端 Transformer。

```figure
spectrogram-window
```

## Build It

### Step 1: 读取与检查

把波形按帧长和 hop 切片，常见语音配置是 25 ms 帧和 10 ms hop。

```python
def frame(signal, frame_len, hop):
    n = 1 + (len(signal) - frame_len) // hop
    return [signal[i * hop : i * hop + frame_len] for i in range(n)]
```

### Step 2: 构建核心表示

给每帧乘 Hann 窗，减少边界不连续带来的频谱泄漏。

```python
import math

def hann(N):
    return [0.5 * (1 - math.cos(2 * math.pi * n / (N - 1))) for n in range(N)]
```

### Step 3: 运行 baseline

对每帧做 FFT，取幅度或功率，得到 STFT 矩阵。

```python
def stft_magnitude(signal, frame_len=400, hop=160):
    win = hann(frame_len)
    frames = frame(signal, frame_len, hop)
    return [magnitudes(dft([w * s for w, s in zip(win, f)])) for f in frames]
```

### Step 4: 升级生产方案

构建 Mel 滤波器组，把线性频率 bin 投影到 Mel bin。

```python
def hz_to_mel(f):
    return 2595.0 * math.log10(1.0 + f / 700.0)

def mel_to_hz(m):
    return 700.0 * (10 ** (m / 2595.0) - 1)

def mel_filterbank(n_mels, n_fft, sr, fmin=0, fmax=None):
    fmax = fmax or sr / 2
    mels = [hz_to_mel(fmin) + (hz_to_mel(fmax) - hz_to_mel(fmin)) * i / (n_mels + 1)
            for i in range(n_mels + 2)]
    hzs = [mel_to_hz(m) for m in mels]
    bins = [int(h * n_fft / sr) for h in hzs]
    fb = [[0.0] * (n_fft // 2 + 1) for _ in range(n_mels)]
    for m in range(n_mels):
        for k in range(bins[m], bins[m + 1]):
            fb[m][k] = (k - bins[m]) / max(1, bins[m + 1] - bins[m])
        for k in range(bins[m + 1], bins[m + 2]):
            fb[m][k] = (bins[m + 2] - k) / max(1, bins[m + 2] - bins[m + 1])
    return fb
```

### Step 5: 验证边界

对 Mel 能量取对数，并加小 epsilon 防止 `log(0)`。

```python
def log_mel(mel_spec, eps=1e-10):
    return [[math.log(max(v, eps)) for v in frame] for frame in mel_spec]
```

### Step 6: 补充步骤

演示 MFCC 的 DCT 步骤，并说明它适合传统分类器而不是 Whisper。

```python
def dct_ii(x, n_coeffs):
    N = len(x)
    return [
        sum(x[n] * math.cos(math.pi * k * (2 * n + 1) / (2 * N)) for n in range(N))
        for k in range(n_coeffs)
    ]
```

## Use It

生产中优先复用目标模型训练时的特征配置。Whisper 是 16 kHz、80-bin log-mel、30 秒窗口。TTS 声码器常固定 24 kHz、hop 256 或 300。分类模型要看预训练 backbone 的配置。不要凭感觉改 `n_mels`、`hop_len` 或归一化。

## Ship It

保存为 `outputs/skill-feature-extractor.md`。这个 skill 帮你根据下游模型选择特征类型、Mel 数、帧长、hop 和归一化策略。

## Exercises

1. 为 16 kHz 语音实现 STFT 幅度矩阵，并打印形状。
2. 把 STFT 投影成 80-bin Mel 频谱图，比较线性频率和 Mel 频率的差异。
3. 为同一段音频生成 log-mel 和 MFCC，解释为什么现代 ASR 更偏好 log-mel。

## Key Terms

| 术语 | 常见说法 | 实际含义 |
|------|----------|----------|
| 频谱图 | 时间频率图 | 分帧 FFT 的幅度或功率矩阵。 |
| Mel 尺度 | 近似人耳感知的频率刻度 | 低频密、高频稀。 |
| Log-mel | 取对数后的 Mel 能量 | 现代 ASR 和 TTS 的默认输入。 |
| Hop | 相邻帧的步长 | 控制时间分辨率和帧数。 |
| 窗口函数 | 帧上的平滑权重 | 减少频谱泄漏。 |
| MFCC | Mel 倒谱系数 | 传统语音特征，不是 Whisper 输入。 |

## Further Reading

- [Davis, Mermelstein (1980). Comparison of parametric representations for monosyllabic word recognition](https://ieeexplore.ieee.org/document/1163420)，the MFCC paper.
- [Stevens, Volkmann, Newman (1937). A Scale for the Measurement of the Psychological Magnitude Pitch](https://pubs.aip.org/asa/jasa/article-abstract/8/3/185/735757/)，the original mel scale.
- [OpenAI，Whisper source, log_mel_spectrogram](https://github.com/openai/whisper/blob/main/whisper/audio.py)，read the reference implementation.
- [librosa feature extraction docs](https://librosa.org/doc/main/feature.html)，reference for `mfcc`, `melspectrogram`, and hop/window.
- [NVIDIA NeMo，audio preprocessing](https://docs.nvidia.com/deeplearning/nemo/user-guide/docs/en/main/asr/asr_all.html#featurizers)，production-scale pipeline for Parakeet + Canary models.
