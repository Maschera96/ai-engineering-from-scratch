# Whisper，架构与微调

> Whisper 是一个 30 秒窗口的 Transformer encoder-decoder，训练于 68 万小时多语种弱监督音频文本对。一个架构覆盖多任务，在 99 种语言上都很稳，是 2026 年 ASR 参考点。

**Type:** Build
**Languages:** Python
**Prerequisites:** Phase 6 · 04（ASR），Phase 5 · 10（注意力），Phase 7 · 05（完整 Transformer）
**Time:** ~75 分钟

## 问题

Whisper 开箱即用很强，但生产部署仍会踩坑：长音频幻觉、语言误判、时间戳偏移、跨 chunk 重复、微调时 tokenizer 或 Mel 管线被改坏。

理解架构和微调边界，能让你知道什么时候用 vanilla Whisper，什么时候用 faster-whisper、whisperx、LoRA 或直接换模型。

## 概念

![Whisper encoder-decoder, tasks, chunked inference, fine-tune](../assets/whisper.svg)

Whisper 输入固定为 30 秒 log-mel，encoder 处理音频，decoder 自回归生成文本和任务 token。

模型变体从 Tiny 到 Large-v3 和 Turbo，质量、速度、显存各不相同。Turbo 是 2026 常用部署折中。

微调可以全量，也可以用 LoRA。多数领域适配更适合冻结 encoder，只调 decoder 或 attention 投影。

不要替换 Whisper tokenizer，不要改 Mel 特征定义，除非你准备重新训练整套模型。

## Build It

### Step 1: 读取与检查

直接运行 Whisper，理解语言检测、任务 token、温度 fallback 和 no-speech threshold。

```
<|startoftranscript|><|en|><|transcribe|><|notimestamps|> Hello world.<|endoftext|>
```

### Step 2: 构建核心表示

对长音频做 chunked 推理。2026 常用 whisperx 获取词级时间戳和对齐。

```python
import whisper
model = whisper.load_model("large-v3-turbo")
result = model.transcribe(
    "clip.wav",
    language="en",
    task="transcribe",
    temperature=0.0,
    condition_on_previous_text=False,  # prevents runaway repetition
)
print(result["text"])
for seg in result["segments"]:
    print(f"[{seg['start']:.2f}–{seg['end']:.2f}] {seg['text']}")
```

### Step 3: 运行 baseline

用 LoRA 做轻量微调，设置 `target_modules`、rank 和冻结策略。

```python
# whisperx is the 2026 reference for long-form with word-level timestamps
import whisperx
model = whisperx.load_model("large-v3-turbo", device="cuda", compute_type="float16")
segments = model.transcribe("1hour.mp3", batch_size=16, chunk_size=30)
```

### Step 4: 升级生产方案

抓取 cross-attention 权重，看 decoder 在不同 step 关注哪些音频帧。

```python
from transformers import WhisperForConditionalGeneration, WhisperProcessor
from peft import LoraConfig, get_peft_model

model = WhisperForConditionalGeneration.from_pretrained("openai/whisper-large-v3-turbo")
lora = LoraConfig(
    r=16, lora_alpha=32, target_modules=["q_proj", "v_proj"],
    lora_dropout=0.1, bias="none", task_type="SEQ_2_SEQ_LM",
)
model = get_peft_model(model, lora)
# model.print_trainable_parameters()  -> ~3M trainable / 809M total
```

## Use It

短音频直接用 faster-whisper 或 whisper.cpp。长音频用 VAD + chunk + whisperx。多说话人会议先 diarize。低延迟流式不一定选 Whisper，Parakeet 或专用 streaming ASR 可能更合适。

## Ship It

保存为 `outputs/skill-whisper-tuner.md`。这个 skill 帮你设计 Whisper 微调或推理流水线。

## Exercises

1. 用同一段音频比较 Tiny、Small 和 Turbo 的 WER 与速度。
2. 给一小时音频加 VAD chunking，检查重复和漏词。
3. 用 5 小时领域数据做 LoRA 微调，并与零样本 Whisper 比较。

## Key Terms

| 术语 | 常见说法 | 实际含义 |
|------|----------|----------|
| Whisper | encoder-decoder ASR 模型 | 使用 30 秒 log-mel 输入。 |
| LoRA | 低秩适配 | 微调少量权重。 |
| no_speech_threshold | 静音阈值 | 控制静音段输出。 |
| temperature fallback | 温度回退 | 解码失败时提高采样温度。 |
| whisperx | 长音频对齐工具 | 常用于词级时间戳。 |

## Further Reading

- [Radford et al. (2022). Whisper paper](https://arxiv.org/abs/2212.04356)，the original architecture and training recipe.
- [OpenAI (2024). Whisper Large-v3-turbo release](https://github.com/openai/whisper/discussions/2363)，4-layer decoder, 8× speedup.
- [Bain et al. (2023). WhisperX](https://arxiv.org/abs/2303.00747)，long-form, word-aligned, diarized.
- [Systran，faster-whisper repo](https://github.com/SYSTRAN/faster-whisper)，CTranslate2-backed, 4× faster.
- [HuggingFace，Whisper fine-tune tutorial](https://huggingface.co/blog/fine-tune-whisper)，canonical LoRA / full-FT walkthrough.
