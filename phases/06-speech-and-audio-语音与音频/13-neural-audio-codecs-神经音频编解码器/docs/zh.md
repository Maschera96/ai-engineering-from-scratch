# 神经音频编解码器，EnCodec、SNAC、Mimi、DAC 与语义声学拆分

> 2026 年音频生成几乎全是 token。EnCodec、SNAC、Mimi 和 DAC 把连续波形变成 Transformer 可预测的离散序列。语义 token 与声学 token 的拆分，是音频 Transformer 之后最重要的架构变化。

**Type:** Learn
**Languages:** Python
**Prerequisites:** Phase 6 · 02（频谱图），Phase 10 · 11（量化），Phase 5 · 19（子词分词）
**Time:** ~60 分钟

## 问题

语言模型擅长预测离散 token，不擅长直接预测 24 kHz 波形。神经 codec 把音频压成离散 codebook 索引，让音频生成变成序列建模。

codec 选择决定 token 数、延迟、重建质量、语义控制和是否适合音乐或语音。

## 概念

![Four codec landscape: EnCodec, DAC, SNAC (multi-scale), Mimi (semantic+acoustic)](../assets/codec-comparison.svg)

核心技巧是 Residual Vector Quantization（RVQ）。第一层 codebook 给粗表示，后续 codebook 逐步补残差。

EnCodec 通用，DAC 高保真但帧率高，SNAC 有层级结构，Mimi 为语音和全双工对话优化。

帧率直接影响 LM 序列长度。86 Hz 乘 8 codebooks 会让 10 秒音频变成数千 token。

语义与声学拆分把第一 codebook 当作内容或语义，后续 codebooks 当作音色和细节，适合 TTS 和语音对话。

## Build It

### Step 1: 读取与检查

用 EnCodec 编码，得到 `(1, n_codebooks, n_frames)` 的 int64 code。

```
frame_t → [semantic_token_t, acoustic_token_0_t, acoustic_token_1_t, ..., acoustic_token_6_t]
```

### Step 2: 构建核心表示

解码并测重建误差或感知质量。

```python
from encodec import EncodecModel
import torch

model = EncodecModel.encodec_model_24khz()
model.set_target_bandwidth(6.0)  # kbps

wav = torch.randn(1, 1, 24000)
with torch.no_grad():
    encoded = model.encode(wav)
codes, scale = encoded[0]
# codes: (1, n_codebooks, n_frames), dtype=int64
```

### Step 3: 运行 baseline

模拟 Mimi 风格的语义声学拆分。

```python
with torch.no_grad():
    wav_recon = model.decode([(codes, scale)])

from torchaudio.functional import compute_deltas
import torch.nn.functional as F

mse = F.mse_loss(wav_recon[:, :, :wav.shape[-1]], wav).item()
```

### Step 4: 升级生产方案

解释为什么 AR LM 能在 codec token 上工作。

```python
from moshi.models import loaders
mimi = loaders.get_mimi()

with torch.no_grad():
    codes = mimi.encode(wav)  # shape (1, 8, frames@12.5Hz)

semantic = codes[:, 0]
acoustic = codes[:, 1:]
```

## Use It

AR TTS 和全双工对话优先 Mimi。音乐高保真可看 DAC 或 SNAC。普通非神经压缩仍然用 Opus。低延迟任务要先算 frame rate 乘 codebooks 带来的 token 量。

## Ship It

保存为 `outputs/skill-codec-picker.md`。这个 skill 帮你为生成或压缩任务选择神经音频 codec。

## Exercises

1. 用 EnCodec 编码 10 秒语音，计算 token 数。
2. 比较 24 kHz 与 48 kHz codec 的重建质量和序列长度。
3. 设计一个 text-to-codec TTS 的 codebook 预测顺序。

## Key Terms

| 术语 | 常见说法 | 实际含义 |
|------|----------|----------|
| 神经 codec | 神经网络音频压缩器 | 把波形变成离散 token。 |
| RVQ | 残差向量量化 | 多 codebook 逐步逼近信号。 |
| codebook | 离散向量表 | 每帧选择一个索引。 |
| Mimi | 语音 codec | Moshi/Hibiki 使用。 |
| 语义声学拆分 | 内容与细节分离 | 提升生成可控性。 |

## Further Reading

- [Défossez et al. (2023). EnCodec](https://arxiv.org/abs/2210.13438)，the RVQ baseline.
- [Kumar et al. (2023). Descript Audio Codec (DAC)](https://arxiv.org/abs/2306.06546)，highest-fidelity open.
- [Siuzdak (2024). SNAC](https://arxiv.org/abs/2410.14411)，multi-scale RVQ.
- [Kyutai (2024). Mimi codec](https://kyutai.org/codec-explainer)，semantic-acoustic split, WavLM distillation.
- [Borsos et al. (2023). AudioLM](https://arxiv.org/abs/2209.03143)，the two-stage semantic/acoustic paradigm.
- [Zeghidour et al. (2021). SoundStream](https://arxiv.org/abs/2107.03312)，the original streamable RVQ codec.
