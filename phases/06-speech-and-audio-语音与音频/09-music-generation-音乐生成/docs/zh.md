# 音乐生成，MusicGen、Stable Audio、Suno 与授权地震

> 2026 年音乐生成中，Suno v5 和 Udio v4 主导商业端，MusicGen、Stable Audio Open 和 ACE-Step 领先开源端。技术问题基本解决，法律问题在 2025 到 2026 年重塑了整个领域。

**Type:** Build
**Languages:** Python
**Prerequisites:** Phase 6 · 02（频谱图），Phase 4 · 10（扩散模型）
**Time:** ~75 分钟

## 问题

音乐生成不只是“给 prompt 出音频”。产品必须回答长度、结构、风格控制、歌声、版权、商业使用、披露和可编辑性。

很多开源模型的许可证不允许商业使用，很多商业模型的输出权利又依赖平台条款。技术选择和授权策略必须一起做。

## 概念

![Music generation: token-LM vs diffusion, the 2026 model map](../assets/music-generation.svg)

一条路线是在神经 codec token 上训练语言模型，MusicGen 属于这类。

另一条路线是在 Mel 或 latent 上做扩散，Stable Audio 属于这类。

商业系统如 Suno、Udio、Lyria 通常是混合方案，含歌词、结构、声乐和后处理。

评估看 FAD、CLAP、人工 MOS 和实际授权风险。

## Build It

### Step 1: 读取与检查

用 MusicGen 生成短音乐片段。

```python
from audiocraft.models import MusicGen
import torchaudio

model = MusicGen.get_pretrained("facebook/musicgen-small")
model.set_generation_params(duration=10)
wav = model.generate(["upbeat synthwave with driving drums, 128 BPM"])
torchaudio.save("out.wav", wav[0].cpu(), 32000)
```

### Step 2: 构建核心表示

加入 melody conditioning 控制旋律。

```python
melody, sr = torchaudio.load("humming.wav")
wav = model.generate_with_chroma(
    ["jazz piano cover"],
    melody.squeeze(),
    sr,
)
```

### Step 3: 运行 baseline

用 FAD 衡量生成音乐和真实音乐分布距离。

```python
from frechet_audio_distance import FrechetAudioDistance
fad = FrechetAudioDistance()

fad.get_fad_score("generated_folder/", "reference_folder/")
```

### Step 4: 升级生产方案

把音乐生成接入 LLM workflow，明确 prompt schema、元数据和披露。

```python
prompt = "Write a 30-second jazz loop. Describe the drums, bass, and piano voicing."
description = llm.complete(prompt)
music = musicgen.generate([description], duration=30)
```

## Use It

需要完整商业权利时优先选择许可证清晰的模型或商业 API。背景音乐可用 MusicGen-large。声乐产品必须处理歌词权利、声音相似、AI 披露和水印。长于 30 秒的音乐要做 chunk、crossfade 或结构规划。

## Ship It

保存为 `outputs/skill-music-designer.md`。这个 skill 帮你选择音乐生成模型、授权策略、长度计划和披露元数据。

## Exercises

1. 用固定 key、BPM 和乐器生成三段背景音乐。
2. 把 30 秒片段 crossfade 成 2 分钟，并听漂移。
3. 为一个商业 app 写授权和 AI 披露清单。

## Key Terms

| 术语 | 常见说法 | 实际含义 |
|------|----------|----------|
| MusicGen | codec-token 音乐模型 | 开源音乐生成基线。 |
| FAD | Fréchet Audio Distance | 音乐生成分布指标。 |
| CLAP | 文本音频对齐模型 | 用于衡量 prompt 匹配。 |
| crossfade | 交叉淡入淡出 | 拼接长音乐的常用技巧。 |
| 披露元数据 | AI 生成标记 | 满足平台和法规要求。 |

## Further Reading

- [Copet et al. (2023). MusicGen](https://arxiv.org/abs/2306.05284)，the open autoregressive benchmark.
- [Evans et al. (2024). Stable Audio Open](https://arxiv.org/abs/2407.14358)，the sound-design default.
- [ACE-Step](https://github.com/ace-step/ACE-Step)，open 4B full-song generator, April 2026.
- [Suno v5 platform docs](https://suno.com)，the commercial quality leader.
- [AudioLDM2](https://arxiv.org/abs/2308.05734)，latent diffusion for music + sound effects.
- [WMG-Suno settlement coverage](https://www.musicbusinessworldwide.com/suno-warner-music-settlement/)，Nov 2025 precedent.
