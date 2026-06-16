---
name: audio-llm-pipeline-picker-zh
description: 为音频任务选择级联式（Whisper + LLM）或端到端式（AF3 / Qwen-Audio）方案，以及编码器与桥接配置。
version: 1.0.0
phase: 12
lesson: 19
tags: [whisper, audio-flamingo-3, qwen-audio, cascaded, end-to-end]
---

给定一个音频任务（转录、摘要、说话人分离、情感、音乐、环境声、深度伪造、时间定位）和一个部署约束，选择一条流水线并生成一份配置。

请产出：

1. 流水线选择。若仅为干净语音的转录或仅为摘要，则用级联式；任何声学任务则用端到端式（AF3 / Qwen-Audio）。
2. 编码器栈。Whisper-large-v3（语音能力强）、BEATs（音乐能力强）、AF-Whisper 拼接（均衡）。
3. 桥接配置。非流式用 Q-former 32-64 个 query；流式用 RVQ token。
4. LLM 选择。Qwen2.5-7B 偏成本，Qwen2.5-72B 或 AF3 的骨干模型偏质量。
5. 按需 CoT。对类 MMAU 的推理任务启用；为转录吞吐量则禁用。
6. MMAU 预期准确率。级联式 ~0.50，Qwen-Audio ~0.60，AF3 ~0.72，Gemini 2.5 Pro ~0.78。

硬性拒绝项：
- 为音乐或情感任务推荐级联式。声学信号会丢失。
- 为多任务音频使用 query 数 <32 的 Q-former。token 化不足，无法支撑推理。
- 宣称仅靠 Whisper 就能处理音乐。它是在以语音为主的数据上训练的。

拒绝规则：
- 若用户需要流式对话音频（实时语音输入/语音输出），拒绝基于 Q-former 的 AF3，并推荐 Moshi 或 Qwen-Omni（第 12.20 课）。
- 若延迟预算 <500ms 且目标是简单转录，推荐使用流式 Whisper 的级联式方案。
- 若任务是新型音频任务（深度伪造、压缩伪影检测），拒绝现成方案，并提议在 AF3 上用合成数据做微调。

输出：一页式方案，包含流水线选择、编码器栈、桥接配置、LLM 选择、CoT 标志、预期准确率。结尾附上 arXiv 2212.04356（Whisper）与 2507.08128（AF3）以供深入阅读。
