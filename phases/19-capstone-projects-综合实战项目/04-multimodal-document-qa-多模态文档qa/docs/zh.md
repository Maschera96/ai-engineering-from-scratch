# Capstone 04 — 多模态文档 QA（视觉优先 PDF、表格、图表）

> 2026 年文档 QA 的前沿已经从 OCR-then-text 转向视觉优先的 late interaction。ColPali、ColQwen2.5 和 ColQwen3-omni 把每个 PDF 页面视为图像，用 multi-vector late interaction 嵌入，并让查询直接关注 patches。在金融 10-K、科学论文和手写笔记上，这种模式大幅胜过 OCR-first。请在 1 万页上端到端构建这条流水线，并发布与 OCR-then-text 的并排对比。

**Type:** Capstone
**Languages:** Python（pipeline）、TypeScript（viewer UI）
**Prerequisites:** Phase 4（computer vision）、Phase 5（NLP）、Phase 7（transformers）、Phase 11（LLM engineering）、Phase 12（multimodal）、Phase 17（infrastructure）
**Phases exercised:** P4 · P5 · P7 · P11 · P12 · P17
**Time:** 30 小时

## 问题

企业手里有大量会被 OCR 流水线弄坏的 PDF：带旋转表格的扫描 10-K、充满公式的科学论文、只有作为图像才有意义的图表、手写批注。把这些材料按 text-first 处理，意味着丢掉一半信号。2026 年的答案是在原始页面图像上做 late-interaction multi-vector retrieval。ColPali（Illuin Tech）引入了它；ColQwen2.5-v0.2 和 ColQwen3-omni 推高了准确率。在 ViDoRe v3 上，vision-first retrieval 明显高于 OCR-then-text，而且差距在图表、表格和手写内容上更大。

代价是存储和延迟。ColQwen embedding 每页约 2048 个 patch vectors，而不是单个 1024-dim vector。原始存储会膨胀。DocPruner（2026）提供 50% pruning，且没有可测量的准确率损失。你要索引 1 万页，度量 ViDoRe v3 nDCG@5，在 2 秒内返回答案，并直接与 OCR-then-text baseline 对比。

## 概念

Late interaction 意味着每个 query token 都会和每个 patch token 打分，然后把每个 query token 的最大分数求和。你获得细粒度匹配，而不需要单个 pooled vector。Multi-vector index（Vespa、Qdrant multi-vector 或 AstraDB）存储 per-patch embeddings，并在检索时运行 MaxSim。

回答器是 vision-language model，它接收查询和 top-k 检索页面图像，并用 evidence regions（bounding boxes 或 page references）写出答案。Qwen3-VL-30B、Gemini 2.5 Pro 和 InternVL3 是 2026 年前沿选择。对公式和科学记号，OCR fallback（Nougat、dots.ocr）会作为可选文本通道拼接进去。

评估是二维矩阵。一条轴是内容类型（纯文本段落、密集表格、柱状/折线图、手写笔记、公式）。另一条轴是检索方法（vision-first late interaction vs OCR-then-text vs hybrid）。每个 cell 都有 nDCG@5 和 answer accuracy。报告就是交付物。

## 架构

```
PDFs -> page renderer (PyMuPDF, 180 DPI)
           |
           v
  ColQwen2.5-v0.2 embed (multi-vector per page, ~2048 patches)
           |
           +------> DocPruner 50% compression
           |
           v
   multi-vector index (Vespa or Qdrant multi-vector)
           |
query ----+----> retrieve top-k pages (MaxSim)
           |
           v
  VLM answerer: Qwen3-VL-30B | Gemini 2.5 Pro | InternVL3
    inputs: query + top-k page images + optional OCR text
           |
           v
  answer with cited page numbers + evidence regions
           |
           v
  Streamlit / Next.js viewer: highlighted boxes on source page
```

## 技术栈

- 页面渲染：PyMuPDF (fitz)，180 DPI，portrait-normalized
- Late-interaction model：ColQwen2.5-v0.2 或 ColQwen3-omni（Hugging Face 上的 vidore team）
- 索引：带 multi-vector field 的 Vespa，或 Qdrant multi-vector，或带 MaxSim 的 AstraDB
- Pruning：DocPruner 2026 policy（保留 high-variance patches，50% compression 且准确率损失 < 0.5%）
- OCR fallback（公式 / 密集表格）：dots.ocr 或 Nougat
- VLM answerer：自托管 Qwen3-VL-30B 或托管 Gemini 2.5 Pro；InternVL3 作为 fallback
- 评估：ViDoRe v3 benchmark，M3DocVQA 用于 multi-page reasoning
- Viewer UI：Next.js 15，用 canvas overlay 显示 evidence regions

## 构建它

1. **摄取。** 遍历包含 10-K、科学论文和扫描文档的 1 万页 PDF 语料。把每页渲染成 1536x2048 PNG。持久化 `{doc_id, page_num, image_path}`。

2. **嵌入。** 对每个页面图像运行 ColQwen2.5-v0.2。输出形状约为 2048 个 dim 128 的 patch embeddings。应用 DocPruner 保留信号最高的一半。写入 Vespa multi-vector field 或 Qdrant multi-vector。

3. **查询。** 对每个传入查询，用 query tower 嵌入（token-level embeddings）。对索引运行 MaxSim：对每个 query token，取页面 patch embeddings 上最大 dot-product，然后求和。返回 top-k pages。

4. **综合。** 用查询和 top-5 页面图像调用 Qwen3-VL-30B。Prompt：“Answer using only the supplied pages. Cite each claim by (doc_id, page) and name the region (figure, table, paragraph).”

5. **证据区域。** 后处理答案，抽取引用区域。如果 VLM 输出 bounding boxes（Qwen3-VL 会这样做），在 viewer 中把它们渲染为 overlays。

6. **OCR fallback。** 对被识别为公式密集的页面（基于 image variance 的 heuristic），运行 Nougat 或 dots.ocr，并把 OCR 文本作为图像旁的额外通道传入。

7. **评估。** 运行 ViDoRe v3（retrieval nDCG@5）和 M3DocVQA（multi-page QA accuracy）。也在同一语料上用同一 synthesizer 运行 OCR-then-text pipeline。生成 content-type × approach 矩阵。

8. **UI。** 先做 Streamlit prototype；再做 Next.js 15 production viewer，带逐页 evidence-region overlay。

## 使用它

```
$ doc-qa ask "what was the 2024 operating margin change for segment EMEA?"
[retrieve]   top-5 pages in 320ms (ColQwen2.5, MaxSim, Vespa)
[synth]      qwen3-vl-30b, 1.4s, cited (form-10k-2024, p. 88) + (..., p. 92)
answer:
  EMEA operating margin moved from 18.2% to 16.8%, a 140bp decline.
  cited: 10-K-2024.pdf p.88 (Table 4, Segment Operating Margin)
         10-K-2024.pdf p.92 (MD&A, Operating Performance)
[viewer]     open with highlighted bounding boxes overlaid on p.88 Table 4
```

## 交付它

`outputs/skill-doc-qa.md` 描述交付物：一个针对特定语料调优的视觉优先多模态文档 QA 系统，并在 ViDoRe v3 上与 OCR-then-text baseline 对比评估。

| Weight | Criterion | How it is measured |
|:-:|---|---|
| 25 | ViDoRe v3 / M3DocVQA 准确率 | 相比 OCR-text baseline 和公开 leaderboard 的 benchmark numbers |
| 20 | Evidence-region grounding | 被引用区域实际包含答案 span 的比例 |
| 20 | 存储和延迟工程 | DocPruner compression ratio、index p95、answer p95 |
| 20 | 多页推理 | 在人工标注的 100-question multi-page set 上的准确率 |
| 15 | Source-inspection UX | Viewer clarity、overlay fidelity、side-by-side comparison tools |
| **100** | | |

## 练习

1. 在同一语料上度量 ColQwen2.5-v0.2 vs ColQwen3-omni。哪个模型能答对另一个漏掉的页面？给索引增加 “content class” tag，按类型路由。

2. 激进 pruning embeddings（75%、90%）。找到 compression cliff：ViDoRe nDCG@5 低于 OCR baseline 的点。

3. 构建 hybrid：并行运行 OCR-then-text 和 ColQwen，用 RRF 融合，再用 cross-encoder 重排。Hybrid 是否胜过任一单独方法？它在哪里最有帮助？

4. 把 Qwen3-VL-30B 换成更小的 VLM（Qwen2.5-VL-7B）。度量 accuracy-per-dollar 曲线。

5. 增加手写笔记支持。渲染 handwriting corpus，用 ColQwen 嵌入，度量 retrieval。与 handwriting OCR pipeline 对比。

## 关键术语

| Term | What people say | What it actually means |
|------|-----------------|------------------------|
| Late interaction | “ColPali 风格检索” | Query tokens 独立地与页面 patches 打分；MaxSim 聚合 |
| Multi-vector | “Per-patch embedding” | 每个文档有多个向量，而不是一个 pooled vector |
| MaxSim | “Late-interaction scoring” | 对每个 query token，取 document vectors 上最大相似度；再求和 |
| DocPruner | “Patch compression” | 2026 年 pruning 方法，以几乎无准确率损失保留 50% patches |
| ViDoRe v3 | “Document-retrieval benchmark” | 2026 年度量 visual-document retrieval 的标准 |
| Evidence region | “Cited bounding box” | 源页面上定位答案 span 的 bbox |
| OCR fallback | “Equation channel” | 与视觉并行使用的文本流水线，处理公式或表格密集页面 |

## 延伸阅读

- [ColPali (Illuin Tech) repository](https://github.com/illuin-tech/colpali) — late-interaction 文档检索参考
- [ColPali paper (arXiv:2407.01449)](https://arxiv.org/abs/2407.01449) — 基础方法论文
- [ColQwen family on Hugging Face](https://huggingface.co/vidore) — 可用于生产的 checkpoints
- [M3DocRAG (Adobe)](https://arxiv.org/abs/2411.04952) — 多页多模态 RAG baseline
- [Vespa multi-vector tutorial](https://docs.vespa.ai/en/colpali.html) — 参考 serving stack
- [Qdrant multi-vector support](https://qdrant.tech/documentation/concepts/vectors/#multivectors) — 备用索引
- [AstraDB multi-vector](https://docs.datastax.com/en/astra-db-serverless/databases/vector-search.html) — 备用托管索引
- [Nougat OCR](https://github.com/facebookresearch/nougat) — 支持公式的 OCR fallback
