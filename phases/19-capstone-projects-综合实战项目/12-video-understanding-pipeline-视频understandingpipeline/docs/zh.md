# Capstone 12 — 视频理解流水线（Scene、QA、Search）

> Twelve Labs 把 Marengo + Pegasus 做成了产品。VideoDB 推出了 CRUD-for-video API。AI2 的 Molmo 2 发布了开放 VLM checkpoints。Gemini 的长上下文可以原生处理数小时视频。TimeLens-100K 把大规模 temporal grounding 定义清楚了。2026 年的典型流水线已经稳定下来，scene segmentation、逐场景 caption + embedding、transcript alignment、多向量索引，以及一个能返回 `(start, end)` timestamps 和 frame previews 的查询。这个 capstone 要求摄取 100 小时视频，打公开 benchmark，并测量在计数题与动作题上的 hallucination。

**Type:** Capstone
**Languages:** Python (pipeline), TypeScript (UI)
**Prerequisites:** Phase 4 (CV), Phase 6 (speech), Phase 7 (transformers), Phase 11 (LLM engineering), Phase 12 (multimodal), Phase 17 (infrastructure)
**Phases exercised:** P4 · P6 · P7 · P11 · P12 · P17
**Time:** 30 hours

## Problem

在 2026 年规模下，长视频 QA 是最消耗带宽的多模态问题。Gemini 2.5 Pro 可以原生读取 2 小时视频，但要把 100 小时视频摄取成一个可查询语料库，仍然需要按 scene 建索引。典型生产形态会把 scene segmentation，通常是 TransNetV2 或 PySceneDetect，和 VLM 的逐场景 captioning，通常是 Gemini 2.5、Qwen3-VL-Max 或 Molmo 2，再加 transcript alignment，通常是带词级时间戳的 Whisper-v3-turbo，结合起来。最后放进一个多向量索引，旁并排存 caption、frame embedding 和 transcript。查询流水线返回答案时，要附带 `(start, end)` timestamps 和 frame previews。

Benchmarks 是公开的，包含 ActivityNet-QA、NeXT-GQA，再加你自己的 100 条自定义查询集合。计数类问题和动作顺序类问题上的 hallucination 是公认最难的失败类型，这个 capstone 会明确对它做测量。

## Concept

摄取阶段会并行运行三条流水线。**Scene segmentation** 把视频切成多个 scenes。**VLM captioning** 为每个 scene 生成一条 caption，并从 keyframe 生成一个 frame embedding。**ASR alignment** 产出词级 timestamps。这三条流按 `(scene_id, time range)` 聚合。每个 scene 都会在多向量索引，Qdrant，中拥有三类向量，caption embedding、keyframe embedding 和 transcript embedding。

在查询阶段，自然语言问题会同时命中这三类向量。结果通过 RRF 合并。接着，一个 temporal-grounding adapter，类似 TimeLens，会在最高分 scene 内进一步细化 `(start, end)` 窗口。最后，VLM synthesizer，通常是 Gemini 2.5 Pro 或 Qwen3-VL-Max，会读取 query、top scenes 和裁剪后的 frames，返回带引用时间戳和 frame preview 的答案。

Hallucination 的测量非常关键。计数问题，比如 “有多少人走进房间？”，和动作问题，比如 “厨师是在搅拌前先倒入吗？”，一向不可靠。你要把这些问题和描述性问题分开报告准确率。

## Architecture

```
video file / URL
      |
      v
PySceneDetect / TransNetV2  (scene segmentation)
      |
      +--- per-scene keyframe --- VLM caption + frame embedding
      |                            (Gemini 2.5 Pro / Qwen3-VL-Max / Molmo 2)
      |
      +--- audio channel --- Whisper-v3-turbo ASR + word timestamps
      |
      v
multi-vector Qdrant: {caption_emb, keyframe_emb, transcript_emb}
      |
query:
  dense queries against all three -> RRF merge -> top-k scenes
      |
      v
TimeLens / VideoITG temporal grounding (refine start/end within scene)
      |
      v
VLM synth: query + top scenes + frame previews
      |
      v
answer + (start, end) timestamps + frame thumbs + citations
```

## Stack

- Scene segmentation: TransNetV2，2024 到 2026 年的先进方案，或 PySceneDetect
- ASR: 通过 faster-whisper 使用带词级 timestamps 的 Whisper-v3-turbo
- VLM captioner + answerer: Gemini 2.5 Pro、Qwen3-VL-Max 或 Molmo 2
- Temporal grounding: 以 TimeLens-100K 训练的 adapter，或 VideoITG
- Index: 支持多向量的 Qdrant，覆盖 caption / frame / transcript
- UI: Next.js 15 + HTML5 video player + scene thumbnails
- Eval: ActivityNet-QA、NeXT-GQA、100 题人工标注自定义集合
- Hallucination benchmark: 对计数类和动作类子集分别做人类标注

## Build It

1. **Ingest walker.** 接受 YouTube URLs 或本地 MP4。必要时降采样到 720p。持久化 `{video_id, file_path}`。

2. **Scene segmentation.** 运行 TransNetV2 或 PySceneDetect，生成 `[{scene_id, start_ms, end_ms, keyframe_path}]`。100 小时目标下，场景数约为 6k 到 8k。

3. **ASR pass.** 在音频上运行 Whisper-v3-turbo，导出词级 timestamps，再按 scene 切分 transcript。

4. **VLM captioning.** 对每个 scene，使用 keyframe 调用 Gemini 2.5 Pro 或 Qwen3-VL-Max，并套一个简短的 caption 模板。输出 caption 和 frame embedding。

5. **Multi-vector index.** 创建一个带三个命名向量的 Qdrant collection。Payload 为 `{video_id, scene_id, start_ms, end_ms, keyframe_url}`。

6. **Query.** 自然语言问题会触发三次 dense query，然后用 reciprocal rank fusion 合并，取 top-k=5 个 scenes。

7. **Temporal grounding.** 在 top scene 上运行 TimeLens 风格的 adapter，把 scene 内的 `(start, end)` 窗口细化出来。

8. **VLM synth.** 调用 Gemini 2.5 Pro，输入 query + top-3 scene clips，图像或短视频片段均可，再加 transcripts。要求输出 `(video_id, start_ms, end_ms)` citations。

9. **Eval.** 运行 ActivityNet-QA 和 NeXT-GQA。再构建一个 100 条查询的自定义集合。报告总体准确率，以及按类别拆分的结果，包含 counting、action 和 descriptive。

## Use It

```
$ video-qa ask --url=https://youtube.com/watch?v=X "how many cars pass the intersection in the first minute?"
[scene]    23 scenes detected
[asr]      transcript complete, 4m12s
[index]    69 vectors written (23 scenes x 3)
[query]    top scene: scene 3 [01:32-01:54], confidence 0.84
[ground]   refined window: [00:12-00:58]
[synth]    gemini 2.5 pro, 1.4s
answer:    5 cars pass the intersection between 00:12 and 00:58.
citations: [scene 3: 00:12-00:58]
          [frame preview at 00:14, 00:27, 00:44, 00:51, 00:57]
```

## Ship It

`outputs/skill-video-qa.md` 是可交付成果。给定一个 YouTube URL 或上传视频，这条流水线会为 scenes 建索引，并返回带时间戳引用的问答结果。

| Weight | Criterion | How it is measured |
|:-:|---|---|
| 25 | Temporal grounding IoU | 在保留的 grounding 集上计算 intersection-over-union |
| 20 | QA accuracy | NeXT-GQA 和自定义 100 条查询 |
| 20 | Ingest throughput | 每花 1 美元能处理多少小时视频 |
| 20 | UI and citation UX | Timestamp links、thumbnail strip、jump-to-frame |
| 15 | Hallucination rate | 分别统计计数类和动作类准确率 |
| **100** | | |

## Exercises

1. 在 captioning 流程中，把 Gemini 2.5 Pro 替换为 Qwen3-VL-Max。对 50 个 scenes 的人工评分样本报告 caption quality 的变化。

2. 把逐场景的 frame embedding 从多向量降成一个 pooled vector。测量 retrieval 的退化程度。

3. 构建 “counting strict” 模式。synthesizer 要为每个被计数实例抽取带时间戳的证据，让用户点击验证。测量用户验证是否能降低 hallucination。

4. Benchmark 摄取成本，对三种 VLM 选择分别统计每美元处理的视频时长。找出最优平衡点。

5. 增加 speaker-diarized transcript，对音频运行 pyannote speaker diarization，并为每位说话人的 transcript 建 embedding。演示 “Alice 提到 X 时说了什么？” 这一类查询。

## Key Terms

| Term | What people say | What it actually means |
|------|-----------------|------------------------|
| Scene segmentation | “Shot detection” | 在镜头边界处把视频切分成多个 scenes |
| Multi-vector index | “Caption + frame + transcript” | 在 Qdrant 中为每种表示单独存命名向量 |
| Temporal grounding | “When exactly did it happen” | 为某个 query answer 细化准确的 `(start, end)` 时间窗口 |
| Frame embedding | “Visual representation” | keyframe 的向量表示，用于 scene 级视觉相似度 |
| RRF fusion | “Reciprocal rank fusion” | 多个排序列表的合并策略，是经典 hybrid retrieval 技巧 |
| Counting hallucination | “Miscount” | VLM 在 “how many X” 问题上的已知失败模式 |
| ActivityNet-QA | “Video-QA benchmark” | 长视频问答准确率 benchmark |

## Further Reading

- [AI2 Molmo 2](https://allenai.org/blog/molmo2) — 开放 VLM checkpoints
- [TimeLens (CVPR 2026)](https://github.com/TencentARC/TimeLens) — 大规模 temporal grounding
- [Gemini Video long-context](https://deepmind.google/technologies/gemini) — 托管参考方案
- [VideoDB](https://videodb.io) — CRUD-for-video API 参考
- [Twelve Labs Marengo + Pegasus](https://www.twelvelabs.io) — 商业参考方案
- [TransNetV2](https://github.com/soCzech/TransNetV2) — scene segmentation model
- [PySceneDetect](https://github.com/Breakthrough/PySceneDetect) — 经典开源替代方案
- [ActivityNet-QA](https://arxiv.org/abs/1906.02467) — 参考评测 benchmark
