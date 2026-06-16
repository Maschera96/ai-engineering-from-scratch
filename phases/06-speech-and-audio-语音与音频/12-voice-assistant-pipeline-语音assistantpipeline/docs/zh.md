# 构建语音助手流水线，Phase 6 综合实战

> 把 lessons 01 到 11 拼起来。构建一个能听、能推理、能说回来的语音助手。2026 年这已经是工程问题，不是研究问题，但集成细节决定它能不能上线。

**Type:** Build
**Languages:** Python
**Prerequisites:** Phase 6 · 04, 05, 06, 07, 11；Phase 11 · 09（函数调用）；Phase 14 · 01（智能体循环）
**Time:** ~120 分钟

## 问题

语音助手是七层系统：麦克风和 chunking、VAD、流式 STT、LLM 与工具、流式 TTS、播放、打断处理。每层都能单独工作，但组合后会暴露延迟、竞态和安全问题。

用户不会容忍半秒以上的尾延迟、错误打断、工具调用沉默或 TTS 念出注入内容。

## 概念

![语音 assistant pipeline: mic → VAD → STT → LLM+tools → TTS → speaker](../assets/voice-assistant.svg)

七个组件必须同时有延迟预算和失败路径。STT 可以和 VAD 重叠，LLM 可以边生成边触发 TTS，但工具调用往往是顺序瓶颈。

三类常见失败：把静音当语音导致幻觉，把用户停顿当结束导致打断，把工具错误暴露成尴尬沉默。

2026 参考栈包括 LiveKit Agents、Pipecat、Vapi、Retell 或自建 WebRTC + STT + LLM + TTS。

## Build It

### Step 1: 读取与检查

用伪代码实现麦克风采集和 chunking。

```python
import sounddevice as sd

def mic_stream(chunk_ms=20, sr=16000):
    q = queue.Queue()
    def cb(indata, frames, time, status):
        q.put(indata.copy().flatten())
    with sd.InputStream(channels=1, samplerate=sr, blocksize=int(sr * chunk_ms/1000), callback=cb):
        while True:
            yield q.get()
```

### Step 2: 构建核心表示

用 VAD 聚合一个用户 turn。

```python
def capture_turn(stream, vad, pre_roll_ms=300, silence_ms=500):
    buf, pre, triggered = [], collections.deque(maxlen=pre_roll_ms // 20), False
    silent = 0
    for chunk in stream:
        pre.append(chunk)
        if vad(chunk):
            if not triggered:
                buf = list(pre)
                triggered = True
            buf.append(chunk)
            silent = 0
        elif triggered:
            silent += 20
            buf.append(chunk)
            if silent >= silence_ms:
                return b"".join(buf)
```

### Step 3: 运行 baseline

串起 streaming STT 到 LLM 再到 streaming TTS。

```python
async def turn(audio_bytes):
    transcript = await stt.transcribe(audio_bytes)
    async for token in llm.stream(transcript):
        async for audio in tts.stream(token):
            await speaker.play(audio)
```

### Step 4: 升级生产方案

在 LLM loop 中加入工具调用和失败回退文本。

```python
tools = [
    {"name": "get_weather", "parameters": {"location": "string"}},
    {"name": "set_timer", "parameters": {"seconds": "int"}},
]

async for chunk in llm.stream(user_text, tools=tools):
    if chunk.type == "tool_call":
        result = dispatch(chunk.name, chunk.args)
        continue_streaming(result)
    if chunk.type == "text":
        await tts.stream(chunk.text)
```

### Step 5: 验证边界

实现用户打断，取消旧的 LLM 和 TTS 任务。

```python
tts_task = asyncio.create_task(tts_loop())
while True:
    chunk = await mic.get()
    if vad(chunk):
        tts_task.cancel()
        await speaker.stop()
        await new_turn()
        break
```

## Use It

消费者应用重视低延迟和自然打断，客服重视合规和观测，无障碍重视可靠性，edge 重视模型大小。必须披露 AI 助手身份，限制原始音频保留，记录每阶段 P95。

## Ship It

保存为 `outputs/skill-voice-assistant-architect.md`。这个 skill 帮你产出完整语音助手规格，包括组件、延迟预算、观测和合规。

## Exercises

1. 为一个邮件语音助手写七层组件表。
2. 把端到端 P95 预算拆到每个阶段。
3. 给工具调用失败设计两次重试和 fallback 话术。

## Key Terms

| 术语 | 常见说法 | 实际含义 |
|------|----------|----------|
| 语音助手 | 听、想、说的系统 | ASR、LLM、TTS 和工具的组合。 |
| VAD | 语音活动检测 | 判断用户是否在说话。 |
| 流式 STT | 边听边转写 | 降低首 token 延迟。 |
| 流式 TTS | 边生成边播放 | 降低首段音频延迟。 |
| 合规披露 | 说明 AI 身份 | 语音产品的基础要求。 |

## Further Reading

- [LiveKit，voice agent quickstart](https://docs.livekit.io/agents/)，production-grade reference.
- [Pipecat，voice agent examples](https://github.com/pipecat-ai/pipecat)，DIY-friendly framework.
- [OpenAI Realtime API](https://platform.openai.com/docs/guides/realtime)，the managed voice-native path.
- [Kyutai Moshi](https://github.com/kyutai-labs/moshi)，full-duplex reference (Lesson 15).
- [Porcupine wake-word](https://picovoice.ai/products/porcupine/)，wake-word gating.
- [Anthropic，tool use guide](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)，LLM function calling.
