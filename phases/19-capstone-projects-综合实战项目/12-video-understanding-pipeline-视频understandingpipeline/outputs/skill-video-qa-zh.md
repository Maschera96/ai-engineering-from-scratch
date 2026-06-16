---
name: video-qa-zh
description: 构建一条视频理解流水线，包含 scene segmentation、multi-vector indexing、temporal grounding 和 timestamped citations。
version: 1.0.0
phase: 19
lesson: 12
tags: [capstone, video, multimodal, gemini, qwen-vl, molmo, transnet, qdrant]
---

给定 100 小时视频，构建摄取流水线和查询系统，用 (start, end) timestamps 加 frame previews 回答自然语言问题。

构建计划：

1. 摄取 videos（YouTube URLs 或 MP4）；必要时降采样到 720p。
2. 用 TransNetV2 或 PySceneDetect 做 scene segmentation；输出 `[{scene_id, start_ms, end_ms, keyframe_path}]`。
3. 用 Whisper-v3-turbo（faster-whisper）做 ASR，生成 word-level timestamps；按 scene 切片。
4. 用 Gemini 2.5 Pro、Qwen3-VL-Max 或 Molmo 2 做 VLM captioning；输出 caption + frame embedding。
5. Qdrant multi-vector index，每个 scene 有三个 named vectors（caption_emb、frame_emb、transcript_emb）和 payload {video_id, scene_id, start_ms, end_ms, keyframe_url}。
6. Query：三路并行 dense queries；用 reciprocal rank fusion 合并；top-k=5 scenes。
7. Temporal grounding（TimeLens adapter 或 VideoITG）在 top scene 内细化 (start, end)。
8. VLM synthesis（Gemini 2.5 Pro），输入 query + top-3 scene clips + transcript；要求 `(video_id, start_ms, end_ms)` citations。
9. 在 ActivityNet-QA、NeXT-GQA，以及 100-query 人工标注 custom set 上评估。报告 overall accuracy 和按 question class（descriptive、counting、action-type）的准确率。

评估标准：

| Weight | Criterion | Measurement |
|:-:|---|---|
| 25 | Temporal grounding IoU | held-out grounding set 上的 IoU |
| 20 | QA accuracy | NeXT-GQA 和 100-query custom set |
| 20 | Ingest throughput | 每美元索引的视频小时数 |
| 20 | UI 和 citation UX | Timestamp links、thumbnail strip、jump-to-frame |
| 15 | Hallucination rate | 单独报告 counting 和 action-type accuracy |

硬性拒收：

- 每个 scene 只 pool 一个向量的 pipelines。必须使用 multi-vector 才能体现类别差异。
- 没有 (start, end) citations 的答案。
- 只报告一个 overall accuracy，却没有 counting/action subset breakdown。
- VLM synthesis 没有直接接收 scene frames（text-only inputs 会丢失视觉 grounding）。

拒绝规则：

- 视频 license provenance 不清楚时，拒绝服务；每个 video_id 都必须有 license tag。
- 拒绝在超过实测 throughput 的 ingest rates 上声称 “real-time” response。
- 拒绝把 counting/action hallucination number 藏在 overall accuracy 数字里。

输出：一个仓库，包含 scene segmentation + ASR + captioning pipeline、multi-vector Qdrant collection、temporal grounding adapter、带 timestamp deep-links 的 Next.js 15 viewer、三项 benchmark eval results（ActivityNet-QA、NeXT-GQA、custom），以及一份说明文档，列出你观察到的三类 counting 或 action-type failure class，以及降低每类问题的 retrieval 或 synthesis change。
