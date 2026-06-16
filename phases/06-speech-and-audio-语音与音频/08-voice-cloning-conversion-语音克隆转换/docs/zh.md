# 语音克隆与语音转换

> 语音克隆用别人的声音读你的文本。语音转换把你的声音改写成别人的声音，同时保留你说的内容。两者都依赖同一个分解：把说话人身份和内容分开。

**Type:** Build
**Languages:** Python
**Prerequisites:** Phase 6 · 06（说话人识别），Phase 6 · 07（TTS）
**Time:** ~75 分钟

## 问题

语音克隆已经从研究 demo 变成产品能力。难点不再只是相似度，而是同意、授权、水印、滥用过滤和跨语言口音迁移。

语音转换和克隆都可以被用于无障碍，也可以被用于欺诈。技术方案必须把安全控制当作核心路径。

## 概念

![语音 cloning vs conversion: factorize, swap speaker, recombine](../assets/voice-cloning.svg)

零样本克隆用几秒参考音频生成相同音色的新语音。语音转换保留源语音内容，把音色换成目标说话人。说话人适配则用更多数据微调模型。

伦理不是附加项。生产系统需要明确录音同意、用途范围、撤销流程、审计日志和水印。

2026 常见模型包括 F5-TTS、VibeVoice、OpenVoice V2、XTTS v2、kNN-VC。

## Build It

### Step 1: 读取与检查

用 recognition-synthesis 做代码层面的分解 demo。

```python
def clone_pipeline(ref_audio, text, target_embedder, tts_model):
    speaker_emb = target_embedder.encode(ref_audio)
    mel = tts_model(text, speaker=speaker_emb)
    return vocoder(mel)
```

### Step 2: 构建核心表示

用 F5-TTS 做零样本克隆，并检查参考文本匹配。

```python
from f5_tts.api import F5TTS
tts = F5TTS()
wav = tts.infer(
    ref_file="rohit_5s.wav",
    ref_text="The quick brown fox jumps over the lazy dog.",
    gen_text="Please add milk and bread to my list.",
)
```

### Step 3: 运行 baseline

用 KNN-VC 做语音转换。

```python
import torch
from knnvc import KNNVC  # 2023 model, https://github.com/bshall/knn-vc
vc = KNNVC.load("wavlm-base-plus")
out_wav = vc.convert(source="my_voice.wav", target_pool=["alice_1.wav", "alice_2.wav"])
```

### Step 4: 升级生产方案

嵌入 AudioSeal 水印，并在发布前检测。

```python
from silentcipher import SilentCipher
sc = SilentCipher(model="2024-06-01")
payload = b"consent_id:abc123;ts:1745353200"
watermarked = sc.embed(wav, sr=24000, message=payload)
detected = sc.detect(watermarked, sr=24000)   # returns payload bytes
```

### Step 5: 验证边界

实现同意 gate，没有同意工件就拒绝生成。

```python
def cloned_inference(text, ref_audio, consent_record):
    assert verify_signature(consent_record), "Signed consent required"
    assert consent_record["speaker_id"] == hash_speaker(ref_audio)
    wav = tts.infer(ref_file=ref_audio, gen_text=text)
    wav = watermark(wav, payload=consent_record["id"])
    return wav
```

## Use It

无障碍和个人语音银行是合理应用。名人、政治人物、未成年人、无授权商业克隆一律拒绝。参考音频至少 3 秒，最好 6 到 30 秒，SNR 要高，去静音且无背景音乐。

## Ship It

保存为 `outputs/skill-voice-cloner.md`。这个 skill 帮你选择克隆方式、同意工件、水印和安全过滤。

## Exercises

1. 为一个无障碍语音银行场景写同意模板。
2. 比较 F5-TTS 与 OpenVoice V2 对参考音频长度的要求。
3. 对克隆输出运行 SECS、CER 和水印检测。

## Key Terms

| 术语 | 常见说法 | 实际含义 |
|------|----------|----------|
| 语音克隆 | 用目标音色读新文本 | 常见于 zero-shot TTS。 |
| 语音转换 | 保留内容改变音色 | 输入和输出都是语音。 |
| SECS | 说话人相似度 | ECAPA embedding cosine。 |
| 水印 | 可检测的生成标记 | 用于合规和溯源。 |
| 同意工件 | 可审计授权记录 | 包含姓名、日期、用途和撤销流程。 |

## Further Reading

- [Chen et al. (2024). F5-TTS](https://arxiv.org/abs/2410.06885)，open-source SOTA zero-shot cloning.
- [Baevski et al. / Microsoft (2023). VALL-E](https://arxiv.org/abs/2301.02111) and [VALL-E 2 (2024)](https://arxiv.org/abs/2406.05370)，neural-codec TTS.
- [Qian et al. (2019). AutoVC](https://arxiv.org/abs/1905.05879)，disentanglement-based voice conversion.
- [Baas, Waubert de Puiseau, Kamper (2023). KNN-VC](https://arxiv.org/abs/2305.18975)，retrieval-based VC.
- [SilentCipher (2024)，Audio Watermarking](https://github.com/sony/silentcipher)，production-ready 32-bit audio watermark.
- [ASVspoof 2025 results](https://www.asvspoof.org/)，detector vs synthesizer arms race, updated 2026.
