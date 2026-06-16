---
name: omni-streaming-budget-zh
description: 为目标 TTFAB 和功能集设计一条 Thinker-Talker 流式语音管线（Qwen-Omni / Moshi / Mini-Omni）的规模。
version: 1.0.0
phase: 12
lesson: 20
tags: [qwen-omni, moshi, mini-omni, streaming, ttfab, thinker-talker]
---

给定一份语音优先的产品规格（目标 TTFAB、麦克风采样率、是否需要视觉、是否双语、是否全双工）和一项算力约束（GPU 等级、预算），设计 Thinker-Talker 管线的规模。

产出：

1. 模型家族选型。Moshi（延迟最佳）、Qwen2.5-Omni（开源功能最佳）、Qwen3-Omni（前沿质量）、Mini-Omni（最简单）。
2. Thinker 和 Talker 的规模。7B Thinker + 200-300M Talker 可实现 <400ms TTFAB。70B+ Thinker 追求质量，但需接受更高的 TTFAB。
3. TTFAB 拆解。逐组件的延迟估算。
4. 双工模式。默认采用带 VAD 轮替的半双工；若产品需要插话回应（backchannel）则采用全双工。
5. 视觉集成。使用带绝对时间戳的 TMRoPE 处理交织的视频帧。
6. 部署形态。根据吞吐需求选择单 GPU 还是拆分部署（Thinker 在 A，Talker 在 B）。

硬性否决：
- 提出 70B 的 Talker。Talker 必须足够小，才能跟上语音 token 的速率。
- 使用非流式语音解码器。会让 TTFAB 急剧膨胀。
- 声称全双工是即插即用的。它需要专门的训练数据。

拒绝规则：
- 如果目标 TTFAB <200ms，在单张 A100 上拒绝任何大于 Moshi 级别（7B 融合）的方案。
- 如果产品需要在流中进行音乐生成，拒绝此架构并推荐独立的音乐管线。
- 如果麦克风采样率为 48kHz 且质量要求严格，标注需要更强的语音编码器；不要盲目降采样。

输出：一页式流式方案，包含模型选型、规模、TTFAB 拆解、双工模式、视觉策略、部署。以 arXiv 2503.20215（Qwen2.5-Omni）、2410.00037（Moshi）结尾。
