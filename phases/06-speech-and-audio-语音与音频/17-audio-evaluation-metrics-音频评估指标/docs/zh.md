# 音频评估，WER、MOS、UTMOS、MMAU、FAD 与开放排行榜

> 不能测量，就不能发布。本课列出 2026 年每类音频任务的指标：ASR 用 WER、CER、RTFx，TTS 用 MOS、UTMOS、SECS、ASR round-trip WER，音频语言用 MMAU 和 LongAudioBench，音乐用 FAD 和 CLAP，说话人用 EER。还包括对比用排行榜。

**Type:** Learn
**Languages:** Python
**Prerequisites:** Phase 6 · 04, 06, 07, 09, 10；Phase 2 · 09（模型评估）
**Time:** ~60 分钟

## 问题

每个音频任务都有多个指标，每个指标测不同轴。用错指标会让 dashboard 很漂亮，生产体验很糟。

2026 的评估清单必须按任务拆开：ASR、TTS、克隆、说话人验证、diarization、分类、音乐、音频语言模型和流式语音到语音。

## 概念

![音频 evaluation matrix — metrics vs tasks vs 2026 leaderboards](../assets/eval-landscape.svg)

ASR 主指标是 WER，辅以 CER、RTFx 和首 token 延迟。评分前要小写、去标点、归一化数字。

TTS 看 MOS 或 UTMOS、SECS、round-trip WER/CER 和 TTFA。克隆必须同时看自然度和说话人相似度。

说话人验证看 EER 和 minDCF。Diarization 看 DER 和 JER。分类看 top-1、mAP、macro F1 和逐类召回。

音乐生成看 FAD、CLAP 和人工听评。音频语言模型看 MMAU-Pro、LongAudioBench、AudioCaps FENSE。流式 S2S 必须报告 P50/P95 延迟。

排行榜包括 Open ASR、TTS Arena、Artificial Analysis Speech、MMAU-Pro、LongAudioBench、VoxCeleb、AudioSet 等。

## Build It

### Step 1: 读取与检查

实现带归一化的 WER。

```python
from jiwer import wer, Compose, ToLowerCase, RemovePunctuation, Strip

transform = Compose([ToLowerCase(), RemovePunctuation(), Strip()])
score = wer(
    truth="Please turn on the lights.",
    hypothesis="please turn on the light",
    truth_transform=transform,
    hypothesis_transform=transform,
)
# ~0.17
```

### Step 2: 构建核心表示

对 TTS 输出跑 ASR round-trip WER。

```python
def ttr_wer(tts_model, asr_model, texts):
    errors = []
    for txt in texts:
        audio = tts_model.synthesize(txt)
        recog = asr_model.transcribe(audio)
        errors.append(wer(truth=txt, hypothesis=recog))
    return sum(errors) / len(errors)
```

### Step 3: 运行 baseline

用 ECAPA embedding 算 voice cloning 的 SECS。

```python
from speechbrain.inference.speaker import EncoderClassifier
sv = EncoderClassifier.from_hparams("speechbrain/spkrec-ecapa-voxceleb")

emb_ref = sv.encode_batch(load_wav("reference.wav"))
emb_clone = sv.encode_batch(load_wav("cloned.wav"))
secs = torch.nn.functional.cosine_similarity(emb_ref, emb_clone, dim=-1).item()
```

### Step 4: 升级生产方案

用 FAD 评估音乐生成分布距离。

```python
from frechet_audio_distance import FrechetAudioDistance
fad = FrechetAudioDistance()
score = fad.get_fad_score("generated_folder/", "reference_folder/")
```

### Step 5: 验证边界

复用 Lesson 6 的代码计算说话人验证 EER。

```python
def eer(same_scores, diff_scores):
    thresholds = sorted(set(same_scores + diff_scores))
    best = (1.0, 0.0)
    for t in thresholds:
        far = sum(1 for s in diff_scores if s >= t) / len(diff_scores)
        frr = sum(1 for s in same_scores if s < t) / len(same_scores)
        if abs(far - frr) < best[0]:
            best = (abs(far - frr), (far + frr) / 2)
    return best[1]
```

## Use It

每次发布都要先选一个主指标，再选 1 到 3 个次指标。延迟不能给平均值，要给 P50/P95/P99。分类不能只给 aggregate，要给逐类。ASR 没有归一化规范就不要报告 WER。音乐不能只给 FAD，必须配人工 MOS。

## Ship It

保存为 `outputs/skill-audio-evaluator.md`。这个 skill 帮你为任何音频模型发布选择指标、benchmark、归一化规则和报告格式。

## Exercises

1. 实现 WER，并比较有无文本归一化的结果。
2. 为一个英文西班牙语 TTS 发布写评估表。
3. 把延迟从平均值改成 P50/P95/P99 报告。

## Key Terms

| 术语 | 常见说法 | 实际含义 |
|------|----------|----------|
| WER | 词错误率 | ASR 主指标。 |
| MOS | 人工主观评分 | TTS 自然度金标准。 |
| UTMOS | 学习式 MOS | 快速估计 TTS 质量。 |
| SECS | 说话人相似度 | 克隆评估核心。 |
| FAD | 音频分布距离 | 音乐生成指标。 |
| MMAU-Pro | 音频理解 benchmark | 评估 LALM。 |
| EER | 等错误率 | 说话人验证和检测指标。 |

## Further Reading

- [jiwer](https://github.com/jitsi/jiwer)，WER/CER library with normalization utilities.
- [UTMOS (Saeki et al. 2022)](https://arxiv.org/abs/2204.02152)，learned MOS predictor.
- [Fréchet Audio Distance (Kilgour et al. 2019)](https://arxiv.org/abs/1812.08466)，the music-gen standard.
- [Open ASR Leaderboard](https://huggingface.co/spaces/hf-audio/open_asr_leaderboard)，2026 live rankings.
- [TTS Arena](https://huggingface.co/spaces/TTS-AGI/TTS-Arena)，human-vote TTS leaderboard.
- [MMAU-Pro benchmark](https://mmaubenchmark.github.io/)，LALM reasoning leaderboard.
- [HEAR benchmark](https://hearbenchmark.com/)，audio SSL benchmarks.
