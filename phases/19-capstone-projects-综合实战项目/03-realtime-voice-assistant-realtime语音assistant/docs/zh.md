# Capstone 03 — 实时语音助手（ASR 到 LLM 到 TTS）
> 感觉合适的语音代理具有低于 800 毫秒的端到端延迟，知道您何时停止说话，处理插入，并且可以在不停止的情况下调用工具。 Retell、Vapi、LiveKit Agents 和 Pipecat 都在 2026 年达到了这一标准。他们使用相同的形状来做到这一点：流式 ASR、转弯检测器、流式 LLM 和流式 TTS，所有这些都通过 WebRTC 进行连接，每一跳都有激进的延迟预算。构建一个，测量 WER 和 MOS 以及错误截止率，并在丢包情况下运行它。
**类型：** 顶点
**语言：** Python（代理 + 管道）、TypeScript（Web 客户端）
**先决条件：** 第 6 阶段（语音和音频）、第 7 阶段（变压器）、第 11 阶段（LLM 工程）、第 13 阶段（工具）、第 14 阶段（代理）、第 17 阶段（基础设施）
**练习阶段：** P6 · P7 · P11 · P13 · P14 · P17
**时间：** 30小时
＃＃ 问题
语音是 2025-2026 年发展最快的 AI UX 类别。技术上限每个季度都会下降。 OpenAI Realtime API、Gemini 2.5 Live、Cartesia Sonic-2、ElevenLabs Flash v3、LiveKit Agents 1.0 和 Pipecat 0.0.70 都可以实现低于 800 毫秒的首次音频输出。酒吧不仅仅是延迟。这是交互的感觉：不会打断用户，不会被打断，从句子中途中断中恢复，在对话中调用工具而不导致音频停顿，在紧张的移动网络中生存。
您无法通过拼接三个 REST 调用来实现这一目标。该架构是端到端的流水线流式传输。构建它，故障模式就会变得可见：一个针对背景电视上的电话音频发射而调整的 VAD、一个等待永远不会出现的标点符号的转弯检测器、一个在发射前缓冲 400 毫秒的 TTS。顶峰是在负载下一次修复这些问题并发布延迟和质量报告。
＃＃ 概念
该管道有五个流阶段：**音频输入**（来自浏览器或 PSTN 的 WebRTC）、**ASR**（从 Deepgram Nova-3 或 fast-whisper 流式传输部分转录）、**转弯检测**（VAD 加上一个小型转弯检测器模型，用于读取部分转录以获取完成提示）、**LLM**（判断转弯完成后立即流式传输令牌）、**TTS**（在内部流式传输音频）第一个 LLM 令牌的约 200 毫秒）。
三个交叉问题。 **打断**：当客服人员正在讲话时用户开始讲话，TTS 取消，ASR 立即接听。 **工具使用**：对话中的功能调用（天气、日历）必须在侧通道上运行，而不会停止音频；如果延迟超过 300 毫秒，代理会预先填充确​​认令牌（“一秒...”）。 **背压**：在数据包丢失的情况下，保留部分转录本，VAD 提高语音门阈值，并且代理避免对未确认的消息进行发言。
测量条是定量的。在 15 dB SNR 的 Hamming VAD 基准上，WER 低于 8%。在 100 次测量的通话中，首次音频输出 p50 低于 800 毫秒。误截率低于3%。 TTS 上的 MOS 高于 4.2。单个 g5.xlarge 上有 50 个并发调用。这些数字就是可交付成果。
＃＃ 建筑学
```
browser / Twilio PSTN
        |
        v
   WebRTC / SIP edge
        |
        v
  LiveKit Agents 1.0  (or Pipecat 0.0.70)
        |
   +----+--------------+--------------+-----------------+
   |                   |              |                 |
   v                   v              v                 v
  ASR              VAD v5         turn-detector     side-channel
(Deepgram         (Silero)          (LiveKit)        tools
 Nova-3 /         speech-gate    completion score    (weather,
 Whisper-v3)      per 20ms        on partials        calendar)
   |                   |              |
   +--------+----------+--------------+
            v
        LLM (streaming)
     GPT-4o-realtime / Gemini 2.5 Flash /
     cascaded Claude Haiku 4.5
            |
            v
        TTS streaming
     Cartesia Sonic-2 / ElevenLabs Flash v3
            |
            v
     audio back to caller
            |
            v
   OpenTelemetry voice traces -> Langfuse
```

＃＃ 堆
- 传输：LiveKit Agents 1.0 (WebRTC) 加上 Twilio PSTN 网关； Pipecat 0.0.70 作为替代框架
- ASR：Deepgram Nova-3（流媒体，第一部分低于 300 毫秒）或更快的 Whisper Whisper-v3-turbo 自托管
- VAD：Silero VAD v5 加上 LiveKit 转动检测器（读取部分转录本的小型变压器）
- LLM：用于紧密集成的 OpenAI GPT-4o-realtime、Gemini 2.5 Flash Live 或级联 Claude Haiku 4.5（流媒体完成、单独的音频路径）
- TTS：Cartesia Sonic-2（最低首字节）、ElevenLabs Flash v3 或用于自托管的开源 Orpheus
- 工具：weather/calendar/booking 的 FastMCP 侧通道；如果工具花费 >300 毫秒，代理会预先发出填充物
- 可观察性：OpenTelemetry 语音跨度、Langfuse 语音轨迹（带音频重放）
- 部署：用于自托管 Whisper + Orpheus 的单个 g5.xlarge (24GB VRAM)；托管 API 可实现最低延迟
## 构建它
1. **WebRTC 会话。** 建立一个 LiveKit 房间和一个传输麦克风音频的 Web 客户端。在服务器上，附加加入房间的代理工作人员。
2. **ASR 流式传输。** 将 20ms PCM 帧馈送到 Deepgram Nova-3（或 GPU 上更快的耳语）。订阅部分和最终成绩单。记录每个部分的延迟。
3. **VAD 和转向检测器。** 在帧流上运行 Silero VAD v5。在语音结束事件中，针对最新的部分文字记录触发 LiveKit 转弯检测器。仅当 VAD 表示沉默 500 毫秒且转弯检测器得分完成> 0.6 时才承诺“转弯完成”。
4. **LLM 流。** 轮流完成后，以正在进行的对话和最终成绩单开始 LLM 通话。流式传输令牌。在第一个令牌处，将其移交给 TTS。
5. **TTS 流。** Cartesia Sonic-2 将音频块流回。第一个块必须在第一个 LLM 令牌后 200 毫秒内离开服务器。向 LiveKit room 发送块；客户端通过 WebRTC 抖动缓冲区进行播放。
6. **打断。** 当 TTS 正在播放时 VAD 检测到新的用户语音，立即取消 TTS 流，丢弃剩余的 LLM 输出，并重新启动 ASR。发布 `tts_canceled` 跨度。
7. **工具侧通道。** 将天气和日历注册为函数调用工具。调用时，同时触发调用；如果它在 300 毫秒内没有解决，让 LLM 发出“一秒钟，让我检查”作为填充；工具返回后继续。
8. **评估安全带。** 记录 100 个呼叫。计算 WER（针对保留的转录本）、错误截止率（TTS 在用户句子中间被取消）、首次音频输出 p50、TTS MOS（人类或 NISQA）以及抖动丢失测试（丢弃 3% 的数据包）。
9. **负载测试。** 使用合成调用方在单个 g5.xlarge 上驱动 50 个并发调用。测量持续的第一音频输出 p95。
## 使用它
```
caller: "what is the weather in tokyo tomorrow"
[asr  ] partial @280ms: "what is the"
[asr  ] partial @540ms: "what is the weather"
[turn ] completion score 0.82 at @820ms; commit
[llm  ] first token @960ms
[tool ] weather.tokyo tomorrow -> 68/52 partly cloudy @1140ms
[tts  ] first audio-out @1040ms: "Tokyo tomorrow will be partly cloudy..."
turn latency: 1040ms user-stop -> audio-out
```

## 发货
`outputs/skill-voice-agent.md` 是可交付成果。给定一个域（客户支持、调度或信息亭），它会建立一个 LiveKit 代理，并将 ASR/VAD/LLM/TTS 管道调整到测量栏。评分细则：
|重量 |标准|如何测量 |
|:-:|---|---|
| 25 | 25端到端延迟| p50 在 100 个录音通话中首次音频输出时间低于 800 毫秒 |
| 20 |轮流品质 | Hamming VAD 基准的错误截断率低于 3% |
| 20 |工具使用正确性 |对话过程中的工具调用可返回正确的数据，而不会导致音频停止 |
| 20 |丢包情况下的可靠性 |注入 3% 丢包后的 WER 和轮流稳定性 |
| 15 | 15评估线束完整性|使用公共配置进行可重复测量 |
| **100** | | |
## 练习
1. 将 Deepgram Nova-3 替换为 g5.xlarge 上更快的 Whisper v3 Turbo。测量延迟和 WER 差距。确定 CPU 与 GPU 决策的重要性。
2. 添加中断仲裁策略：当用户在工具调用期间插入时，代理会做什么？比较三种策略（硬取消、完成工具然后停止、下一回合排队）。
3. 运行对抗性转弯检测器测试：让用户在句子中间长时间停顿。调整 VAD 静音阈值和转弯检测器分数阈值，以在不超过 900 毫秒的情况下实现最低的错误切断。
4. 通过 Twilio 在 PSTN 上部署相同的代理。将 PSTN 第一个音频输出与 WebRTC 进行比较。解释抖动缓冲区和编解码器的差异。
5. 添加非英语语言（日语、西班牙语）的语音活动检测。测量 Silero VAD v5 误触发率与特定于语言的微调。
## 关键术语
|术语 |人们怎么说|它实际上意味着什么 ||------|-----------------|------------------------|
|转弯检测| “发言结束” |分类器，在给定 VAD 静音和部分文字记录的情况下，决定用户已完成讲话 |
|打断 | “中断处理” |当 VAD 检测到新用户语音时取消 TTS 播放中途 |
|第一个音频输出 | “延迟” |从用户停止说话到第一个音频数据包离开服务器的时间 |
|虚拟AD | “言语门” |将音频帧分类为语音与静音的模型； Silero VAD v5 是 2026 年默认设置 |
|抖动缓冲器| “音频平滑” |客户端缓冲区可短暂保存数据包以吸收网络差异 |
|填料| “确认令牌”|当工具速度缓慢时，代理会发出简短的短语以避免沉默 |
| MOS | “平均意见得分”|感知语音质量评级； NISQA 是自动化代理 |
## 进一步阅读
- [LiveKit Agents 1.0](https://github.com/livekit/agents) — 参考 WebRTC 代理框架
- [Pipecat](https://github.com/pipecat-ai/pipecat) — 替代的 Python 优先流代理框架
- [OpenAI Realtime API](https://platform.openai.com/docs/guides/realtime) — 集成语音模型参考
- [Deepgram Nova-3 文档](https://developers.deepgram.com/docs) — 流式 ASR 参考
- [Silero VAD v5](https://github.com/snakers4/silero-vad) — VAD 参考模型
- [Cartesia Sonic-2](https://docs.cartesia.ai) — 低延迟 TTS 参考
- [Retell AI 架构](https://docs.retellai.com) — 生产语音代理架构
- [Vapi.ai 生产堆栈](https://docs.vapi.ai) — 备用生产参考