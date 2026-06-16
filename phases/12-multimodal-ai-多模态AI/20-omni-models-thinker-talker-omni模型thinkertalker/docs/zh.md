# Omni 模型：Qwen2.5-Omni 与 Thinker-Talker 拆分

> GPT-4o 在 2024 年 5 月的产品演示之所以具有颠覆性，并不是因为底层模型，而是因为产品形态——一个语音界面：你说话，模型看到摄像头看到的画面，并在 250ms 以内用语音回应你。开源生态在 2024 和 2025 年的剩余时间里一直在追赶这个产品界面。Qwen2.5-Omni（2025 年 3 月）是参考性的开源设计：一个 Thinker（大型文本生成 transformer）加上一个 Talker（并行的语音生成 transformer），由流式语音 token 连接。Mini-Omni 简化了它，Moshi 达到了它的延迟水平，GLM-4-Voice 把它扩展到了中文。本课解读 Thinker-Talker 架构，以及让流式实时对话得以实现的延迟预算。

**类型：** Build
**语言：** Python（标准库，流式管线延迟模拟器 + VAD 循环）
**先修：** Phase 12 · 19（音频 LLM）、Phase 12 · 16（任意到任意）
**时长：** ~180 分钟

## 学习目标

- 把推理管线拆分为 Thinker（文本推理）和 Talker（语音合成），并解释为什么并行流式可行。
- 逐组件计算一次对话交互的首字节音频时间（TTFAB）预算。
- 描述 TMRoPE 在 Thinker 内部跨视觉、音频和文本的时间对齐位置编码。
- 说出三种实时对话模式的名称：半双工、轮流发言、全双工。

## 问题

一个实时语音助手必须快速完成很多事情：

1. 听用户说话。实时语音 token 化、语音活动检测（VAD）以判断用户是否说完。
2. 可选地看。摄像头以 2-4 FPS 输入，与音频一起流入 Thinker。
3. 思考。基于对话历史构造一个回应。
4. 说话。合成音频 token，解码为波形，流式传输到用户的扬声器。

每一步都增加延迟。要有对话感，整个往返必须 < 500ms——低于这个值，用户就不再注意到延迟。GPT-4o 声称 ~250ms。Moshi ~160ms。Qwen2.5-Omni ~350-500ms。

每个组件都需要流式处理。任何环节都不能是"先批量处理一切再解码"。

## 概念

### Thinker 与 Talker

Qwen2.5-Omni 的分解：

- Thinker：一个 7B-80B 的文本生成 transformer。消费交错的文本 + 图像 + 音频 token。输出表示"要说什么"的文本 token。
- Talker：一个更小的语音生成 transformer（200M-1B）。消费 Thinker 的文本输出 token 加上近期的语音上下文 token。输出离散的语音 token（residual-VQ 索引）。
- 语音解码器：一个流式波形解码器（SNAC、MoVQGAN 系列），实时把语音 token 转为音频采样。

这种分离很重要。Thinker 必须够大才能有好的推理能力。Talker 可以很小，因为它的工作是局部的——把文本转为语音 token。更大的 Talker 并不更具表现力，只是更慢。

并行运行两者：

1. Thinker 发出文本 token t_i。
2. Talker（通过流式）消费 t_i 并发出语音 token s_i、s_{i+1}、……、s_{i+k}。
3. 语音解码器在语音 token 到达时逐个消费并发出音频采样。
4. 当 Thinker 处理到文本 token t_{i+3} 时，Talker 已经为 t_0..t_{i+2} 流式输出了音频。

### TMRoPE——时间对齐的多模态位置

Thinker 需要整合图像帧（比如以 4 FPS 到达）、音频帧（以 50 帧/秒到达）和来自对话历史的文本。一个朴素的序列顺序（先所有图像，再所有音频，再文本）会丢失时间对齐。

TMRoPE 为每个 token 分配绝对时间戳。视觉 token 在 t=2.3s。音频 token 在 t=2.32s。用户说"stop"的文本 token 在 t=2.35s。RoPE 按时间戳旋转注意力；模型把它们视为在时间上并发的。

这是让"他一边挥手一边说你好"得以成立的基础设施——模型在同一个概念时刻看到视频帧和音频。

### 流式语音合成

语音 token 必须流式。Mini-Omni（Xie & Wu，2024）提出了"语言模型可以边听边说、边思考边流式"：Thinker 输出 token 和 Talker 输出 token 在同一序列中交错。Thinker 一旦提交下一个文本 token，Talker 就立即触发。没有批次边界。

Moshi（Défossez 等，2024 年 10 月）是最快的开源实现。单张 A100 上 160ms 的 TTFAB。架构：一个单一的 7B transformer，在交替位置发出文本和语音 token，并带有一个"内心独白"，把思考流和说话流分离开。这实际上是把 Thinker + Talker 通过精心训练融合进一个模型。

### VAD 与轮流发言

语音活动检测在输入侧运行。两种模式：

- 半双工：用户说话，模型听。模型说话，用户听。通过 VAD 静默检测（~200ms）实现清晰的交接。
- 全双工：双方可以同时说话。模型可以附和（"嗯哼"）或打断。难得多。Moshi 支持这个。

Qwen2.5-Omni 默认支持半双工，通过静默阈值实现轮流发言。全双工需要应用层处理。

### Qwen3-Omni（2025 年 11 月）

继任者。Qwen3-80B Thinker，更大的 Talker，改进的 TMRoPE-v2。延迟接近 GPT-4o 的 250ms。开放权重。在 OmniBench 上的基准成绩与 Gemini 2.0 Live 相当。

### 生产环境延迟预算

对于一次典型的流式交互：

- 麦克风 -> 音频 token：40-80ms。
- 预填充（提示词 + 历史）：7B 上 100-200ms，70B 上要多得多。
- 第一个 Thinker 文本 token：40ms。
- Talker 处理第一个文本 token：20ms。
- 第一批语音 token 提交：40ms。
- Residual-VQ 解码：30ms。
- 语音波形解码：50-80ms。

总 TTFAB：7B 上 320-510ms，70B 上 600-900ms。前沿质量通常意味着 70B+；因此存在前沿延迟差距。

### Token 速率计算

在 16kHz 语音、50 Hz 基础语音 token 的情况下，每秒输出需要 50 个语音 token。Talker 必须发出 ≥50 tok/s 才能跟上。在 H100 上典型的 LLM 吞吐量为 30-80 tok/s 时，一个小型（200-300M）的 Talker 足够快；一个 7B 的 Talker 会跟不上。

这就是为什么存在专用的小型 Talker 模型，而不是"直接用主模型"。

## 使用它

`code/main.py`：

- 用模拟的 token 发出速率来模拟一个 Thinker-Talker 管线。
- 为可配置的模型尺寸和麦克风采样率计算 TTFAB。
- 用 VAD 静默阈值演示半双工轮流发言。

## 交付它

本课产出 `outputs/skill-omni-streaming-budget.md`。给定一个实时语音产品的目标 TTFAB 和功能集（视觉输入、双语、全双工），它会在 Qwen2.5-Omni、Qwen3-Omni、Moshi 或 Mini-Omni 中做出选择，并确定 Thinker/Talker 的尺寸。

## 练习

1. 你的目标 TTFAB 是 300ms。在一个 7B Thinker 和 300M Talker 上，写出每个组件的延迟。

2. Qwen2.5-Omni 使用 TMRoPE。描述对于这样一个提示词，模型看到了什么：用户在 t=1s 开始说话，摄像头在 t=1.2s 捕捉到一个手势。

3. 全双工支持要求模型在听的同时发出音频。提出一种训练数据格式来教会这一点。

4. 阅读 Moshi 论文第 4 节。描述"内心独白"分离，以及为什么它避免了 Thinker-Talker 拆分。

5. 计算吞吐量预算：要跟上 16kHz 语音、50 个基础层 token/秒，Talker 必须以多快的速度发出 token？

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|-----------------|------------------------|
| Thinker | "推理大脑" | 大型文本生成 transformer，产出要说什么 |
| Talker | "语音生成嘴巴" | 小型 transformer，从 Thinker 的文本产出离散语音 token |
| TTFAB | "延迟预算" | 首字节音频时间：从用户说话结束到第一个音频采样输出 |
| TMRoPE | "时间对齐的 RoPE" | 使用跨视觉、音频、文本的绝对时间戳的位置编码 |
| 半双工 | "轮流发言" | 用户和模型交替；VAD 静默检测用户说完 |
| 全双工 | "同时" | 模型可以同时说和听；具备附和能力 |
| 内心独白 | "Moshi 分离" | 单模型设计，思考流和说话流交错 |

## 延伸阅读

- [Xu et al. — Qwen2.5-Omni (arXiv:2503.20215)](https://arxiv.org/abs/2503.20215)
- [Qwen Team — Qwen3-Omni (arXiv:2509.17765)](https://arxiv.org/html/2509.17765v1)
- [Xie & Wu — Mini-Omni (arXiv:2408.16725)](https://arxiv.org/abs/2408.16725)
- [Défossez et al. — Moshi (arXiv:2410.00037)](https://arxiv.org/abs/2410.00037)
- [Zeng et al. — GLM-4-Voice (arXiv:2412.02612)](https://arxiv.org/abs/2412.02612)
