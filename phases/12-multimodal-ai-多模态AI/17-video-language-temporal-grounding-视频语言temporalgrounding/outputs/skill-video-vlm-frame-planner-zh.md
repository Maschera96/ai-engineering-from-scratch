---
name: video-vlm-frame-planner-zh
description: 为视频语言模型部署规划帧采样、逐帧池化、输出格式与基准目标。
version: 1.0.0
phase: 12
lesson: 17
tags: [video-vlm, temporal-grounding, tmrope, dynamic-fps, benchmarks]
---

给定一个视频任务（动作识别、temporal grounding、摘要、监控、智能体工作流回放）以及一个部署约束（模型上下文、延迟预算、吞吐量），输出一份帧采样与输出方案。

产出：

1. 帧采样器选择。内容稳定用 uniform，动态混合运动用 dynamic-FPS，动作密集用 event-driven，电影化内容用 keyframe+context。
2. 逐帧池化。高细节用 2x2，默认 3x3，智能体工作流用 4x4 或 6x6——此时内容密度不如覆盖范围重要。
3. 时序编码。Qwen2.5-VL 系列用 TMRoPE；较小模型用学习式时序嵌入；单片段任务不做编码。
4. 输出格式。grounding 用带 `{event, start, end, confidence}` 的 JSON；摘要用自由文本；混合流程用 token 分隔。
5. 基准方案。通用用 VideoMME，grounding 用 TempCompass，长程用 EgoSchema。指定预期的准确率档位。
6. 上下文 / 延迟预算。总 token 数 = duration * fps * tokens_per_frame。若超过上下文的 40% 则发出警告。

硬性拒绝：
- 为动作密集视频提议 uniform 采样。会丢失峰值事件。
- 声称 token 分隔输出在下游解析上与 JSON 准确率相当。JSON 更稳健。
- 为任何 2026 年起步的项目推荐 Video-LLaMA。较旧的架构已不再有竞争力。

拒绝规则：
- 如果 duration > 10 分钟且 context < 32k，拒绝并推荐分层摘要或智能体式检索（第 12.18 课）。
- 如果目标准确率为前沿水平（在 VideoMME 上与 Gemini 2.5 Pro 相差 2 分以内），拒绝开源 7B 模型，要求 32B+ 或专有模型。
- 如果在 7B 上对 > 30s 片段的 dynamic-FPS 目标 > 8，从延迟角度拒绝并推荐更低的上限。

输出：一页帧方案，包含采样器、池化、时序编码、输出格式、基准目标、上下文估算。结尾附上 arXiv 2502.13923（Qwen2.5-VL）和 2306.02858（Video-LLaMA）作为对比阅读材料。
