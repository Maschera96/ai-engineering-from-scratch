# 流式语音到语音，Moshi、Hibiki 与全双工对话

> 2024 到 2026 年重新定义了语音 AI。Moshi 用一个模型同时听和说，延迟 200 ms。Hibiki 逐 chunk 做语音到语音翻译。两者都放弃 ASR → LLM → TTS 流水线，改用 Mimi codec token 上的统一全双工架构。

**Type:** Learn
**Languages:** Python
**Prerequisites:** Phase 6 · 13（神经音频 codec），Phase 6 · 11（实时音频），Phase 7 · 05（完整 Transformer）
**Time:** ~75 分钟

## 问题

传统语音智能体有 300 到 500 ms 的延迟地板：VAD 触发，STT 处理，LLM 思考，TTS 生成。每一段都有自己的最小延迟。

Moshi 问了另一个问题：如果没有流水线会怎样？一个模型直接吃音频并连续吐出音频，文本只是内部独白，而不是必经阶段。

## 概念

![Moshi architecture: two parallel Mimi streams + inner-monologue text](../assets/moshi-hibiki.svg)

Moshi 输入两个 Mimi codec 流：用户音频和 Moshi 自己生成的音频。7B Temporal Transformer 同时处理两个音频流和文本内部独白流。

每 80 ms，模型消费用户 Mimi token、消费自己上一步的 Mimi token、生成下一个文本 token，并通过 depth transformer 生成下一个 Moshi Mimi token。

内部独白文本让模型更容易保持语义一致，也免费给出 transcript。

Hibiki 使用相似架构做流式语音翻译。Hibiki-Zero 用句级数据和 GRPO 做延迟优化，减少对词级对齐数据的依赖。

Sesame CSM 类似使用 Mimi head，但它是单向 TTS，不是全双工对话模型。

## Build It

### Step 1: 读取与检查

理解 WebSocket 接口：输入 80 ms Mimi 编码音频，返回 80 ms Mimi 编码音频。

```python
import asyncio
import websockets
from moshi.client_utils import encode_audio_mimi, decode_audio_mimi

async def moshi_chat():
    async with websockets.connect("ws://localhost:8998/api/chat") as ws:
        mic_task = asyncio.create_task(stream_mic_to(ws))
        spk_task = asyncio.create_task(stream_from_to_speaker(ws))
        await asyncio.gather(mic_task, spk_task)
```

### Step 2: 构建核心表示

实现全双工 loop，同时读用户流、写模型流，并让模型能听见自己说了什么。

```python
async def stream_mic_to(ws):
    async for chunk_80ms in mic_stream_at_12_5_hz():
        mimi_tokens = encode_audio_mimi(chunk_80ms)
        await ws.send(serialize(mimi_tokens))

async def stream_from_to_speaker(ws):
    async for msg in ws:
        mimi_tokens, text_token = deserialize(msg)
        audio = decode_audio_mimi(mimi_tokens)
        await play(audio)
```

### Step 3: 运行 baseline

理解训练目标：文本内部独白损失加 codec token 损失。

### Step 4: 升级生产方案

比较 Moshi 擅长的低延迟开放对话和不擅长的工具调用企业智能体。

## Use It

低于 250 ms 的自然对话或语言陪练适合全双工模型。需要复杂工具调用、检索、合规审计的企业智能体仍更适合传统流水线或混合方案。Moshi 的语言覆盖和并发成本也要单独评估。

## Ship It

保存为 `outputs/skill-duplex-pipeline.md`。这个 skill 帮你在全双工架构和传统语音智能体流水线之间做选择。

## Exercises

1. 画出 Moshi 的三条流：用户音频、模型音频、内部文本。
2. 估算 10k DAU 下 Moshi 并发 GPU 成本。
3. 为需要工具调用的语音智能体设计 duplex + sidecar LLM 混合架构。

## Key Terms

| 术语 | 常见说法 | 实际含义 |
|------|----------|----------|
| 全双工 | 边听边说 | 模型说话时仍能听用户。 |
| Moshi | 全双工语音对话模型 | 基于 Mimi token。 |
| Hibiki | 流式语音翻译模型 | 逐 chunk 输出目标语音。 |
| 内部独白 | 并行文本流 | 提升语义一致性。 |
| depth transformer | 帧内 codebook 预测器 | 顺序预测多个 codebook。 |

## Further Reading

- [Défossez et al. (2024). Moshi，speech-text foundation model](https://arxiv.org/html/2410.00037v2)，the paper.
- [Kyutai Labs (2026). Hibiki-Zero](https://arxiv.org/abs/2602.12345)，streaming translation without aligned data.
- [Sesame (2025). Crossing the uncanny valley of voice](https://www.sesame.com/research/crossing_the_uncanny_valley_of_voice)，CSM spec.
- [Kyutai，Moshi repo](https://github.com/kyutai-labs/moshi)，install + server.
- [OpenAI，Realtime API](https://platform.openai.com/docs/guides/realtime)，closed commercial peer.
- [Kyutai，Delayed Streams Modeling](https://github.com/kyutai-labs/delayed-streams-modeling)，the STT/TTS framework under the hood.
