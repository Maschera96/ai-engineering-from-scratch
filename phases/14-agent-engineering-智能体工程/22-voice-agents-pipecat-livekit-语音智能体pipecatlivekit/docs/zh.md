# 语音智能体：Pipecat 与 LiveKit

> 语音智能体在 2026 年是一类一流的生产级品类。Pipecat 为你提供基于帧的 Python 流水线（VAD → STT → LLM → TTS → 传输）。LiveKit Agents 通过 WebRTC 将 AI 模型与用户连接起来。在高端技术栈中，生产环境的延迟目标落在端到端 450–600ms。

**类型：** 学习
**语言：** Python（标准库）
**前置知识：** 阶段 14 · 01（智能体循环）、阶段 14 · 12（工作流模式）
**时间：** 约 60 分钟

## 学习目标

- 描述 Pipecat 基于帧的流水线：DOWNSTREAM（源→汇）和 UPSTREAM（控制）。
- 说出标准的语音流水线各阶段，以及 Pipecat 支持哪些传输方式。
- 解释 LiveKit Agents 的两种语音智能体类（MultimodalAgent、VoicePipelineAgent）及各自适用的场景。
- 总结 2026 年生产环境的延迟预期，以及它如何驱动架构选型。

## 问题所在

语音智能体并不是给文本循环硬塞一个 TTS。延迟预算极其苛刻（约 600ms），部分音频是常态，轮次检测本身是一个模型，传输方式从电话 SIP 到 WebRTC 不一而足。你要么构建一个基于帧的流水线（Pipecat），要么依托一个平台（LiveKit）。

## 概念

### Pipecat (pipecat-ai/pipecat)

- 基于帧的 Python 流水线框架。
- `Frame` → `FrameProcessor` 链。
- 两个流向：
  - **DOWNSTREAM** — 源 → 汇（音频输入，TTS 输出）。
  - **UPSTREAM** — 反馈与控制（取消、指标、抢话）。
- `PipelineTask` 通过事件（`on_pipeline_started`、`on_pipeline_finished`、`on_idle_timeout`）以及用于指标/追踪/RTVI 的观察者来管理生命周期。

典型流水线：

```
VAD (Silero) → STT → LLM (context alternates user/assistant) → TTS → transport
```

传输方式：Daily、LiveKit、SmallWebRTCTransport、FastAPI WebSocket、WhatsApp。

Pipecat Flows 增加了结构化对话（状态机）。Pipecat Cloud 是托管运行时。

### LiveKit Agents (livekit/agents)

- 通过 WebRTC 将 AI 模型与用户连接起来。
- 关键概念：`Agent`、`AgentSession`、`entrypoint`、`AgentServer`。
- 两种语音智能体类：
  - **MultimodalAgent** — 通过 OpenAI Realtime 或同类方案进行直接音频处理。
  - **VoicePipelineAgent** — STT → LLM → TTS 级联；提供文本层面的控制。
- 通过 transformer 模型实现语义轮次检测。
- 原生 MCP 集成。
- 通过 SIP 支持电话。
- 通过 LiveKit Inference 无需 API key 即可使用 50+ 模型；通过插件再支持 200+ 模型。

### 商业平台

Vapi（在优化后的高端技术栈上约 450–600ms）和 Retell（在 180 次测试通话中端到端约 600ms）都构建在这些方案之上。当你想要一个托管的语音技术栈、又不想组建 WebRTC 团队时，就选择平台。

### 这个模式容易出错的地方

- **没有处理抢话。** 用户打断了；智能体却还在继续说。这在 Pipecat 中需要 UPSTREAM 取消帧，在 LiveKit 中需要等价机制。
- **忽略 STT 置信度。** 把低置信度的转写当作金科玉律喂给 LLM。应基于置信度设门控，或请求确认。
- **TTS 句中截断。** 当流水线在话语中途取消时，TTS 需要知道这一点，或者直接切断音频。
- **忽略延迟预算。** 每个组件都会增加 50–200ms。在交付前先把你的链路总和算清楚。

### 2026 年的典型延迟

- VAD：20–60ms
- STT 部分结果：100–250ms
- LLM 首个 token：150–400ms
- TTS 首段音频：100–200ms
- 传输 RTT：30–80ms

端到端 450–600ms 属于高端水平。800–1200ms 很常见。任何超过 1500ms 的延迟都会让人觉得坏掉了。

## 动手构建

`code/main.py` 是一个基于帧的玩具流水线，包含：

- `Frame` 类型（audio、transcript、text、tts_audio、control）。
- 带有 `process(frame)` 的 `Processor` 接口。
- 一个五阶段流水线（VAD → STT → LLM → TTS → transport），以脚本化处理器实现。
- 一个 UPSTREAM 取消帧，用于演示抢话。

运行它：

```
python3 code/main.py
```

该追踪展示了正常流程，以及一个在话语中途停止 TTS 的抢话取消。

## 如何使用

- **Pipecat** 用于完全控制 —— 自定义处理器、Python 优先、可插拔的提供方。
- **LiveKit Agents** 用于以 WebRTC 为先的部署和电话场景。
- **Vapi / Retell** 用于托管语音智能体，无需组建 WebRTC 团队。
- **OpenAI Realtime / Gemini Live** 用于直接的音频输入/音频输出（MultimodalAgent）。

## 交付上线

`outputs/skill-voice-pipeline.md` 搭建一个 Pipecat 形态的语音流水线，包含 VAD + STT + LLM + TTS + transport，外加抢话处理。

## 练习

1. 给你的玩具流水线添加一个指标观察者：统计每个阶段每秒的帧数。延迟在哪里累积？
2. 实现置信度门控的 STT：低于阈值时，请求“能再说一遍吗？”
3. 添加语义轮次检测：用简单规则 —— 如果转写以“?”结尾，则视为轮次结束。
4. 阅读 Pipecat 的传输文档。把标准库传输换成 SmallWebRTCTransport 配置（桩实现）。
5. 在同一个查询上对比 OpenAI Realtime 与 STT+LLM+TTS 级联的表现。文本层面的控制带来了多大的延迟代价？

## 关键术语

| 术语 | 人们的说法 | 实际含义 |
|------|----------------|------------------------|
| Frame | “事件” | 流水线中带类型的数据单元（音频、转写、文本、控制） |
| Processor | “流水线阶段” | 带有 process(frame) 的处理器 |
| DOWNSTREAM | “正向流” | 源到汇：音频输入，语音输出 |
| UPSTREAM | “反馈流” | 控制：取消、指标、抢话 |
| VAD | “语音活动检测” | 检测用户何时在说话 |
| 语义轮次检测 | “智能轮次结束判定” | 基于模型判断用户是否说完 |
| MultimodalAgent | “直接音频智能体” | 音频输入、音频输出；中间没有文本 |
| VoicePipelineAgent | “级联智能体” | STT + LLM + TTS；文本层面的控制 |

## 延伸阅读

- [Pipecat docs](https://docs.pipecat.ai/getting-started/introduction) —— 基于帧的流水线、处理器、传输
- [LiveKit Agents docs](https://docs.livekit.io/agents/) —— WebRTC + 语音原语
- [Vapi](https://vapi.ai/) —— 托管语音平台
- [Retell AI](https://www.retellai.com/) —— 托管语音，已做延迟基准测试
