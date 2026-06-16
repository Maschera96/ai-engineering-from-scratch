# 语音识别（ASR），CTC、RNN-T 与 Attention

> 语音识别是在每个时间步做音频分类，再用懂语言和静音的序列模型把结果粘起来。CTC、RNN-T 和 attention 是三条主路。选择其中一种，并理解原因。

**Type:** Build
**Languages:** Python
**Prerequisites:** Phase 6 · 02（频谱图与 Mel），Phase 5 · 08（文本 CNNs 与 RNNs），Phase 5 · 10（注意力）
**Time:** ~45 分钟

## 问题

ASR 的输入是连续音频，输出是离散文本，二者长度不同且没有逐帧对齐。模型必须同时解决声学识别、对齐、语言约束和静音处理。

2026 年你通常不会从零训练 ASR，但你必须理解 CTC、RNN-T、attention、解码和 WER，否则无法排查幻觉、延迟和流式错误。

## 概念

![Three ASR formulations: CTC, RNN-T, attention-encoder-decoder](../assets/asr-formulations.svg)

CTC 允许模型在每帧预测字符、子词或 blank，再通过折叠重复和移除 blank 得到文本。它简单、快，适合流式和端侧。

RNN-T 把声学编码器和预测网络结合，天然适合流式 ASR。Parakeet-TDT 属于这一类。

Attention encoder-decoder 直接从音频特征生成文本，质量高但流式更难。Whisper 使用这一范式。

WER 是主指标：`(S + D + I) / N`。评分前必须做文本归一化。

## Build It

### Step 1: 读取与检查

实现 greedy CTC decode，折叠重复 token 并删除 blank。

```python
def ctc_greedy(frame_logits, blank=0, vocab=None):
    # frame_logits: list of per-frame probability vectors
    preds = [max(range(len(p)), key=lambda i: p[i]) for p in frame_logits]
    out = []
    prev = -1
    for p in preds:
        if p != prev and p != blank:
            out.append(p)
        prev = p
    return "".join(vocab[i] for i in out) if vocab else out
```

### Step 2: 构建核心表示

实现 beam-search CTC，保留多个候选并比较分数。

```python
def ctc_beam(frame_logits, beam=8, blank=0):
    import math
    beams = [([], 0.0)]  # (tokens, log_prob)
    for p in frame_logits:
        log_p = [math.log(max(pi, 1e-10)) for pi in p]
        candidates = []
        for seq, lp in beams:
            for t, lpt in enumerate(log_p):
                new = seq[:] if t == blank else (seq + [t] if not seq or seq[-1] != t else seq)
                candidates.append((new, lp + lpt))
        candidates.sort(key=lambda x: -x[1])
        beams = candidates[:beam]
    return beams[0][0]
```

### Step 3: 运行 baseline

手写 WER，理解替换、删除、插入三种错误。

```python
def wer(ref, hyp):
    r, h = ref.split(), hyp.split()
    dp = [[0] * (len(h) + 1) for _ in range(len(r) + 1)]
    for i in range(len(r) + 1):
        dp[i][0] = i
    for j in range(len(h) + 1):
        dp[0][j] = j
    for i in range(1, len(r) + 1):
        for j in range(1, len(h) + 1):
            cost = 0 if r[i - 1] == h[j - 1] else 1
            dp[i][j] = min(
                dp[i - 1][j] + 1,
                dp[i][j - 1] + 1,
                dp[i - 1][j - 1] + cost,
            )
    return dp[len(r)][len(h)] / max(1, len(r))
```

### Step 4: 升级生产方案

调用 Whisper 做推理，观察 chunk、语言和 no-speech 设置对结果的影响。

```python
import whisper
model = whisper.load_model("large-v3-turbo")
result = model.transcribe("clip.wav")
print(result["text"])
```

### Step 5: 验证边界

对流式场景，比较 Parakeet、wav2vec 2.0 和 Whisper-streaming 的延迟取舍。

```python
from transformers import pipeline
asr = pipeline("automatic-speech-recognition", model="nvidia/parakeet-tdt-1.1b")
for chunk in streaming_audio():
    print(asr(chunk, return_timestamps=True))
```

## Use It

离线高质量多语种默认 Whisper-large-v3-turbo。开源低延迟流式可看 Parakeet-TDT 或 Moonshine。端侧、小模型或低资源语言要单独评估。长音频必须加 VAD 和 chunk 策略，静音片段要测幻觉率。

## Ship It

保存为 `outputs/skill-asr-picker.md`。这个 skill 帮你为部署目标选择 ASR 模型、解码策略、chunking、VAD 和评估方案。

## Exercises

1. 给一段 CTC logits 实现 greedy decode。
2. 实现简化 beam search，并比较 beam width 对结果和速度的影响。
3. 用归一化前后的文本各算一次 WER，说明差异。

## Key Terms

| 术语 | 常见说法 | 实际含义 |
|------|----------|----------|
| ASR | 自动语音识别 | 把语音转换成文本。 |
| CTC | 无显式对齐的序列损失 | 通过 blank 和折叠处理长度差。 |
| RNN-T | 流式转录模型 | 适合低延迟 ASR。 |
| WER | 词错误率 | ASR 主指标。 |
| VAD | 语音活动检测 | 用于过滤静音和切分语音。 |

## Further Reading

- [Graves et al. (2006). Connectionist Temporal Classification](https://www.cs.toronto.edu/~graves/icml_2006.pdf)，the CTC paper.
- [Graves (2012). Sequence Transduction with RNNs](https://arxiv.org/abs/1211.3711)，the RNN-T paper.
- [Radford et al. / OpenAI (2022). Whisper: Robust Speech Recognition via Large-Scale Weak Supervision](https://arxiv.org/abs/2212.04356)，the 2022 canonical paper; v3-turbo extension in 2024.
- [NVIDIA NeMo，Parakeet-TDT card](https://huggingface.co/nvidia/parakeet-tdt-1.1b)，2026 Open ASR Leaderboard leader.
- [Hugging Face，Open ASR Leaderboard](https://huggingface.co/spaces/hf-audio/open_asr_leaderboard)，live benchmark across 25+ models.
