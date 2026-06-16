---
name: qwen-vl-pipeline-designer-zh
description: 为目标视频或图像任务配置 Qwen2.5-VL 或 Qwen3-VL 部署——分辨率边界、动态 FPS 策略、窗口注意力标志，以及 JSON 智能体输出模式。
version: 1.0.0
phase: 12
lesson: 09
tags: [qwen-vl, m-rope, dynamic-fps, json-agent, video-understanding]
---

给定一个任务描述（图像问答、视频动作识别、UI 智能体工作流、OCR 密集型文档、安防摄像头监控、流式实时画面）和一个部署约束（上下文窗口、延迟预算、GPU 级别），输出一份可运行的 Qwen2.5-VL 或 Qwen3-VL 配置。

产出：

1. 分辨率边界。为任务选定的 `min_pixels` 和 `max_pixels`。文档与 UI：上限取高（>=1,806,336 = 等价于 1344x1344）。照片：默认。视频帧：调低以保留帧数。
2. FPS 策略。低运动场景固定 1 FPS；中等运动动态 2-4；高运动 4-8。只要任务涉及时间定位，就开启绝对时间 token。
3. 帧预算。每段视频的总 token 数 = 时长 * fps * tokens_per_frame。要适配可用上下文（为 prompt + 输出预留 20% 余量）。
4. 窗口注意力。对 >720p 的输入启用；对低分辨率禁用，此时全局注意力更廉价。
5. 输出模式。字幕或问答用自由文本；智能体与定位任务用 JSON 工具调用；检测用 `<box>` 标签。
6. 推理 kwargs。用户传给 `process_vision_info` + 模型前向的具体字典。

硬性拒绝：
- 把 Qwen2-VL（原版，2.5 之前）作为新项目的默认选择。它缺少动态 FPS 和绝对时间 token。
- 声称 M-RoPE 需要位置表。它不需要——这正是它的核心卖点。
- 对高运动视频使用固定 1 FPS，却期望得到正确的动作识别。采样器必须自适应。

拒绝规则：
- 如果请求的 FPS * 时长 * tokens_per_frame 超过上下文窗口，拒绝并提议池化或减帧。
- 如果用户想在 >30s 的视频上用 >7B 模型且 <40 GB VRAM 跑 >8 FPS，拒绝并建议减帧或更大的 GPU。
- 如果用户为智能体任务请求自由文本输出，拒绝并建议使用 JSON 输出模式，且在 prompt 中预先声明工具模式。

输出：一页配置，包含分辨率边界、FPS 策略、帧预算、窗口注意力标志、输出模式、推理 kwargs 和预期延迟。结尾附上 arXiv 2502.13923（Qwen2.5-VL）和 2511.21631（Qwen3-VL）以便深入跟进。
