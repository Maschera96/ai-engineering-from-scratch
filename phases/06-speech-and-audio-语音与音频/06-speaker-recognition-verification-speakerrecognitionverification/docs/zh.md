# 说话人识别与验证

> ASR 问“说了什么？”说话人识别问“是谁说的？”数学看起来一样，都是 embedding 加 cosine，但生产决策几乎都卡在一个 EER 数字上。

**Type:** Build
**Languages:** Python
**Prerequisites:** Phase 6 · 02（频谱图与 Mel），Phase 5 · 22（Embedding 模型）
**Time:** ~45 分钟

## 问题

说话人任务分三类：验证某人是不是注册用户，识别一段音频属于谁，或者把会议按说话人分段。它们都依赖说话人 embedding，但阈值、注册协议和评估完全不同。

语音克隆让简单 cosine 验证变得危险。任何欺诈级语音认证都必须加 anti-spoof 前端。

## 概念

![Enrollment + verification pipeline with embedding + cosine + EER](../assets/speaker-verification.svg)

说话人 embedding 把一段音频压成固定维向量。ECAPA-TDNN、x-vector、WavLM-SV、ReDimNet 都属于这一类。

验证通常计算注册向量和测试向量的 cosine 或 PLDA 分数，再用阈值决定接受或拒绝。

EER 是 FAR 等于 FRR 时的错误率。生产更关心某个 FAR 下的 FRR 或 minDCF。

Diarization 是“谁在什么时候说话”，通常需要 VAD、分段 embedding、聚类和重分割。

## Build It

### Step 1: 读取与检查

用 MFCC 统计量构造玩具 embedding。

```python
def embed_mfcc_stats(signal, sr):
    frames = featurize_mfcc(signal, sr, n_mfcc=13)
    mean = [sum(f[i] for f in frames) / len(frames) for i in range(13)]
    std = [
        math.sqrt(sum((f[i] - mean[i]) ** 2 for f in frames) / len(frames))
        for i in range(13)
    ]
    return mean + std  # 26-d
```

### Step 2: 构建核心表示

实现 cosine similarity 和阈值判定。

```python
def cosine(a, b):
    dot = sum(x * y for x, y in zip(a, b))
    na = math.sqrt(sum(x * x for x in a))
    nb = math.sqrt(sum(x * x for x in b))
    return dot / (na * nb) if na and nb else 0.0

def verify(enroll, test, threshold=0.75):
    return cosine(enroll, test) >= threshold
```

### Step 3: 运行 baseline

从正负样本分数计算 EER。

```python
def eer(same_scores, diff_scores):
    thresholds = sorted(set(same_scores + diff_scores))
    best = (1.0, 1.0, 0.0)  # (fa, fr, threshold)
    for t in thresholds:
        fr = sum(1 for s in same_scores if s < t) / len(same_scores)
        fa = sum(1 for s in diff_scores if s >= t) / len(diff_scores)
        if abs(fa - fr) < abs(best[0] - best[1]):
            best = (fa, fr, t)
    return (best[0] + best[1]) / 2, best[2]
```

### Step 4: 升级生产方案

用 SpeechBrain 或 pyannote 跑生产级注册与验证。

```python
from speechbrain.pretrained import EncoderClassifier

clf = EncoderClassifier.from_hparams(source="speechbrain/spkrec-ecapa-voxceleb")

# enroll: average the embeddings of 3-5 clean samples
enroll = torch.stack([clf.encode_batch(load(x)) for x in enrollment_clips]).mean(0)
# verify
score = clf.similarity(enroll, clf.encode_batch(load("test.wav"))).item()
verdict = score > 0.25   # ECAPA typical threshold; tune on your data
```

### Step 5: 验证边界

用 pyannote 做 diarization，观察重叠语音和短片段的错误。

```python
from pyannote.audio import Pipeline

pipe = Pipeline.from_pretrained("pyannote/speaker-diarization-3.1")
diarization = pipe("meeting.wav", num_speakers=None)
for turn, _, speaker in diarization.itertracks(yield_label=True):
    print(f"{turn.start:.1f}–{turn.end:.1f}  {speaker}")
```

## Use It

语音认证默认 ECAPA 或 WavLM-SV，并在目标通道上调阈值。Diarization 默认 pyannote。注册至少 3 到 5 段干净样本，和部署通道匹配。报告 EER 时必须说明数据集、通道和片段时长。

## Ship It

保存为 `outputs/skill-speaker-verifier.md`。这个 skill 帮你设计说话人验证或 diarization 流水线。

## Exercises

1. 用玩具 embedding 区分两个说话人。
2. 画出 FAR/FRR 随阈值变化的曲线，并找到 EER。
3. 测试同一说话人在手机、耳机和会议麦克风上的阈值漂移。

## Key Terms

| 术语 | 常见说法 | 实际含义 |
|------|----------|----------|
| 说话人 embedding | 固定维声纹向量 | 用于比较身份。 |
| EER | 等错误率 | FAR 等于 FRR 的点。 |
| FAR | 误接受率 | 把冒名者接受为真用户。 |
| FRR | 误拒绝率 | 把真用户拒绝。 |
| Diarization | 说话人分离 | 判断谁在什么时候说话。 |

## Further Reading

- [Snyder et al. (2018). X-Vectors: Robust DNN Embeddings for Speaker Recognition](https://www.danielpovey.com/files/2018_icassp_xvectors.pdf)，the classic deep-embedding paper.
- [Desplanques et al. (2020). ECAPA-TDNN](https://arxiv.org/abs/2005.07143)，dominant architecture 2020–2026.
- [Chen et al. (2022). WavLM: Large-Scale Self-Supervised Pre-Training for Full Stack Speech Processing](https://arxiv.org/abs/2110.13900)，SSL backbone for SV and diarization.
- [Bredin et al. (2023). pyannote.audio 3.1](https://github.com/pyannote/pyannote-audio)，production diarization + embedding stack.
- [VoxCeleb leaderboard (updated 2026)](https://www.robots.ox.ac.uk/~vgg/data/voxceleb/)，current EER standings across models.
