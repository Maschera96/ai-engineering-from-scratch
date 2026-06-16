# 音频语言模型，Qwen2.5-Omni、Audio Flamingo、GPT-4o Audio

> 2026 年音频语言模型能同时理解语音、环境声和音乐。Qwen2.5-Omni-7B 在 MMAU-Pro 上接近 GPT-4o Audio，Audio Flamingo Next 在 LongAudioBench 上超过 Gemini 2.5 Pro。开源和闭源差距基本关闭，除了多音频任务，大家都接近随机。

**Type:** Learn
**Languages:** Python
**Prerequisites:** Phase 6 · 04（ASR），Phase 12 · 03（视觉语言模型），Phase 7 · 10（音频 Transformer）
**Time:** ~45 分钟

## 问题

很多团队会把所有音频任务都交给一个 LALM，但这经常比专用模型更差。转写要 Whisper，分类要 BEATs，diarization 要 pyannote。LALM 的价值在跨模态推理，而不是替代每个专家。

长音频、多音频比较和细粒度时间定位仍然困难。部署前必须按任务轴验证。

## 概念

![音频-language model: 音频 encoder + projector + LLM decoder](../assets/alm-architecture.svg)

常见模板是音频 encoder、投影层和 LLM。音频先变成表示，再映射到语言模型 token 空间。

2026 模型图包括 Qwen2.5-Omni、Qwen3-Omni、SALMONN、Audio Flamingo、AF-Next、LTU、GAMA、Gemini 2.5 Pro、GPT-4o Audio。

MMAU-Pro、LongAudioBench、AudioCaps、ClothoAQA 分别覆盖不同能力。

LALM 适合“听完并推理”的任务，如判断客服是否读了披露语，不适合替代高精度 ASR。

## Build It

### Step 1: 读取与检查

查询 Qwen2.5-Omni，观察文本和语音输出模式。

```python
from transformers import AutoModelForCausalLM, AutoProcessor

processor = AutoProcessor.from_pretrained("Qwen/Qwen2.5-Omni-7B")
model = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-Omni-7B", torch_dtype="auto")

audio, sr = load_wav("clip.wav", sr=16000)
messages = [{
    "role": "user",
    "content": [
        {"type": "audio", "audio": audio},
        {"type": "text", "text": "What sounds do you hear, and what's happening?"},
    ],
}]
inputs = processor.apply_chat_template(messages, tokenize=True, return_tensors="pt")
output = model.generate(**inputs, max_new_tokens=200)
print(processor.decode(output[0], skip_special_tokens=True))
```

### Step 2: 构建核心表示

实现 projector pattern，把音频 embedding 映射到 LLM hidden size。

```python
import torch.nn as nn

class AudioProjector(nn.Module):
    def __init__(self, audio_dim=1280, llm_dim=4096):
        super().__init__()
        self.down = nn.Linear(audio_dim, llm_dim)
        self.act = nn.GELU()
        self.up = nn.Linear(llm_dim, llm_dim)

    def forward(self, audio_features):
        return self.up(self.act(self.down(audio_features)))
```

### Step 3: 运行 baseline

跑 MMAU 或 LongAudioBench 子集，按 speech、sound、music、多音频分别看分数。

```python
from datasets import load_dataset
mmau = load_dataset("MMAU/MMAU-Pro")

correct = 0
for item in mmau["test"]:
    answer = call_model(item["audio"], item["question"], item["choices"])
    if answer == item["correct_choice"]:
        correct += 1
print(f"Accuracy: {correct / len(mmau['test']):.3f}")
```

## Use It

如果任务是转写，先用 Whisper。如果任务是音频问答或合规推理，可用 LALM 读转写或直接读音频。超过 10 分钟的输入先 diarize 和切片。多音频比较必须验证 MMAU-Pro multi-audio 分数。

## Ship It

保存为 `outputs/skill-alm-picker.md`。这个 skill 帮你选择音频语言模型、benchmark 子集、输出模态和 guardrails。

## Exercises

1. 用同一段音频问 LALM 和 Whisper，看转写差异。
2. 实现一个音频 embedding 到文本模型 hidden size 的投影层。
3. 为长电话审计任务设计 LALM 和专用 ASR 的混合流水线。

## Key Terms

| 术语 | 常见说法 | 实际含义 |
|------|----------|----------|
| LALM | 音频语言模型 | 对音频和文本做联合推理。 |
| MMAU-Pro | 音频理解 benchmark | 含 speech、sound、music、多音频。 |
| LongAudioBench | 长音频 benchmark | 测试长上下文音频理解。 |
| Projector | 投影层 | 把音频表示接到 LLM。 |
| 多音频 | 多个音频输入比较 | 2026 仍然薄弱。 |

## Further Reading

- [Chu et al. (2024). Qwen2-Audio](https://arxiv.org/abs/2407.10759)，reference architecture.
- [Alibaba (2025). Qwen2.5-Omni](https://huggingface.co/Qwen/Qwen2.5-Omni-7B)，speech-in-speech-out.
- [NVIDIA (2025). Audio Flamingo 3](https://arxiv.org/abs/2507.08128)，the open long-audio leader.
- [NVIDIA (2026). Audio Flamingo Next](https://arxiv.org/abs/2604.10905)，LongAudioBench SOTA.
- [Tang et al. (2023). SALMONN](https://arxiv.org/abs/2310.13289)，dual-encoder pioneer.
- [MMAU-Pro leaderboard](https://mmaubenchmark.github.io/)，live 2026 rankings.
