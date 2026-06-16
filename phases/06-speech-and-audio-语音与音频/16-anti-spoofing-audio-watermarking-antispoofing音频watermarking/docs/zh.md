# 语音反欺骗与音频水印，ASVspoof 5、AudioSeal、WaveVerify

> 语音克隆比防御走得更快。2026 年生产语音系统需要两件事：检测器，如 AASIST、RawNet2，用于判断真假语音；水印，如 AudioSeal，用于承受压缩和编辑。两者都要有，否则不要发布语音克隆。

**Type:** Build
**Languages:** Python
**Prerequisites:** Phase 6 · 06（说话人识别），Phase 6 · 08（语音克隆）
**Time:** ~75 分钟

## 问题

三类防御相关但不同：anti-spoofing 判断音频是真实还是合成；音频水印把不可感知信号嵌进生成音频；认证溯源用 C2PA 等签名元数据记录来源。

检测应对不合作的攻击者。水印处理合规和自家生成内容。溯源适合文件链路，但元数据可被剥离。生产需要组合防御。

## 概念

![Anti-spoofing vs watermarking vs provenance — three defense layers](../assets/spoofing-watermark.svg)

ASVspoof 5 是 2024 到 2025 的关键 benchmark，包含众包数据、约 2000 个说话人、32 种攻击算法和 CM/SASV 两条赛道。

AASIST 用谱特征上的图注意力，是 anti-spoof 常见强模型。RawNet2 用原始波形卷积前端和 TDNN，是简单基线。NeXt-TDNN 加 WavLM 特征代表 2025 变体。

AudioSeal 是 Meta 的水印默认选择，可逐帧检测，生成器和检测器联合训练，能抗压缩、EQ、速度轻微变化和噪声，检测速度很快。

WavMark 是较早开放基线，但同步慢，不够实时。WaveVerify 针对反转、变速等时间编辑补强。

pitch shift 仍是 2026 水印的通用弱点，所以水印不能替代检测。

## Build It

### Step 1: 读取与检查

写一个玩具 spectral-feature detector，提取谱滚降等特征。

```python
def spectral_rolloff(spec, percentile=0.85):
    cum = 0
    total = sum(spec)
    if total == 0:
        return 0
    threshold = total * percentile
    for k, v in enumerate(spec):
        cum += v
        if cum >= threshold:
            return k
    return len(spec) - 1

def is_suspicious(audio):
    spec = magnitude_spectrum(audio)
    rolloff = spectral_rolloff(spec)
    return rolloff / len(spec) > 0.92
```

### Step 2: 构建核心表示

用 AudioSeal 嵌入和检测水印，检查 presence probability 和 16-bit payload。

```python
from audioseal import AudioSeal
import torch

generator = AudioSeal.load_generator("audioseal_wm_16bits")
detector = AudioSeal.load_detector("audioseal_detector_16bits")

audio = load_wav("generated.wav", sr=16000)[None, None, :]
payload = torch.tensor([[1, 0, 1, 1, 0, 1, 0, 0, 1, 1, 0, 1, 0, 1, 1, 0]])
watermark = generator.get_watermark(audio, sample_rate=16000, message=payload)
watermarked = audio + watermark

result, decoded_payload = detector.detect_watermark(watermarked, sample_rate=16000)
# result: float in [0, 1] — probability of watermark presence
# decoded_payload: 16 bits; match against embedded payload
```

### Step 3: 运行 baseline

实现 EER 评估，理解真假分数的阈值选择。

```python
def eer(real_scores, fake_scores):
    thresholds = sorted(set(real_scores + fake_scores))
    best = (1.0, 0.0)
    for t in thresholds:
        far = sum(1 for s in fake_scores if s >= t) / len(fake_scores)
        frr = sum(1 for s in real_scores if s < t) / len(real_scores)
        if abs(far - frr) < best[0]:
            best = (abs(far - frr), (far + frr) / 2)
    return best[1]
```

### Step 4: 升级生产方案

把检测、水印、C2PA 和审计日志放进生产路径。

```python
def safe_tts(text, voice, clone_reference=None):
    if clone_reference is not None:
        verify_consent(user_id, clone_reference)
    audio = tts_model.synthesize(text, voice)
    audio_with_wm = audioseal_embed(audio, payload=build_payload(user_id, model_id))
    manifest = c2pa_sign(audio_with_wm, user_id, timestamp=now())
    return audio_with_wm, manifest
```

## Use It

语音生成必须有 AudioSeal 或等价水印。语音认证必须有 anti-spoof 检测，不能只靠声纹 cosine。阈值不能只在 ASVspoof 2019 上调完就上线，必须做真实通道校准。

## Ship It

保存为 `outputs/skill-spoof-defender.md`。这个 skill 帮你为语音生成或语音认证部署选择检测模型、水印、溯源 manifest 和运营手册。

## Exercises

1. 用 toy detector 区分真实和合成样本，并画 ROC。
2. 给生成语音嵌入 16-bit payload，再经过 MP3 压缩后检测。
3. 为银行 IVR 设计反欺骗和 liveness challenge。

## Key Terms

| 术语 | 常见说法 | 实际含义 |
|------|----------|----------|
| anti-spoofing | 反欺骗检测 | 判断语音是否伪造。 |
| AudioSeal | 音频水印模型 | 可嵌入并检测 payload。 |
| ASVspoof | 反欺骗 benchmark | 评估真假语音检测。 |
| EER | 等错误率 | 检测器常用指标。 |
| C2PA | 内容溯源 manifest | 签名记录来源元数据。 |

## Further Reading

- [Todisco et al. (2024). ASVspoof 5](https://dl.acm.org/doi/10.1016/j.csl.2025.101825)，the current benchmark.
- [Defossez et al. (2024). AudioSeal](https://arxiv.org/abs/2401.17264)，the watermark default.
- [Chen et al. (2025). WaveVerify](https://arxiv.org/abs/2507.21150)，MoE detector for temporal attacks.
- [Jung et al. (2022). AASIST](https://arxiv.org/abs/2110.01200)，the SOTA detection backbone.
- [AudioMarkBench (2024)](https://proceedings.neurips.cc/paper_files/paper/2024/file/5d9b7775296a641a1913ab6b4425d5e8-Paper-Datasets_and_Benchmarks_Track.pdf)，robustness evaluation.
- [C2PA specification](https://c2pa.org/specifications/specifications/)，provenance manifest format.
