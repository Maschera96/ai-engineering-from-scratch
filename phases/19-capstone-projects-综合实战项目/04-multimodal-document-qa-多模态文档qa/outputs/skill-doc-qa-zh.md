---
name: doc-qa-zh
description: 在 1 万页文档上构建视觉优先的多模态文档 QA 系统，使用 late-interaction 检索和证据区域引用。
version: 1.0.0
phase: 19
lesson: 04
tags: [capstone, multimodal, rag, colpali, colqwen, late-interaction, pdf]
---

给定一组 PDF 语料（10-K 年报、科学论文、扫描文档），构建一条流水线：用 ColPali 风格的 late interaction 把页面作为图像索引，并用页面级证据区域回答问题。

构建计划：

1. 用 PyMuPDF 以 180 DPI 将每个 PDF 页面渲染为 1536x2048 PNG。
2. 用 ColQwen2.5-v0.2 或 ColQwen3-omni 嵌入每个页面。将 multi-vector patch embeddings 存入 Vespa、Qdrant multi-vector 或 AstraDB。
3. 应用 DocPruner 风格的 50% patch pruning。在 ViDoRe v3 上验证准确率下降低于 0.5%。
4. 查询时：嵌入查询 tokens；对每个页面的 patches 计算 MaxSim；排序 top-k。
5. 用 Qwen3-VL-30B 或 Gemini 2.5 Pro 综合答案，输入查询和 top-5 页面图像。要求引用 `(doc_id, page, region)` 锚点。
6. 对公式或表格密集页面，可选运行 Nougat 或 dots.ocr 作为文本通道，并与图像一起输入。
7. 构建 Next.js 15 viewer，把证据区域作为 bounding boxes 叠加到源页面上。
8. 在 ViDoRe v3 和 M3DocVQA 上评估。生成 content-class × approach 矩阵，对比 vision-first 与 OCR-then-text 在纯文本、表格、图表、手写和公式上的表现。

评估标准：

| Weight | Criterion | Measurement |
|:-:|---|---|
| 25 | ViDoRe v3 / M3DocVQA 准确率 | 在匹配页面上与 OCR-then-text baseline 做 benchmark |
| 20 | 证据区域 grounding | 被引用区域中实际包含答案 span 的比例 |
| 20 | 存储和延迟工程 | DocPruner 压缩、索引 p95、答案 p95 低于 2 秒 |
| 20 | 多页推理 | 在人工标注的 100 问多页集合上的准确率 |
| 15 | 来源检查 UX | 叠加层保真度、对比工具、逐页 explorer |

硬性拒收：

- 把 OCR 文本改装进单向量 embedding，却声称是 “vision-first” 的 OCR-first pipeline。
- 任何丢弃 patch-level bounding boxes、因此无法渲染证据叠加层的系统。
- 报告存储数据却没有记录 DocPruner 设置。

拒绝规则：

- 没有专门脱敏策略时，拒绝索引扫描版法律合同。ColQwen embeddings 会泄漏内容。
- 拒绝服务用户未披露的语料查询。受监管领域必须有 audit trail。
- 如果没有在同一语料上同时运行两条流水线，拒绝与 OCR-then-text 做比较。

输出：一个仓库，包含摄取流水线、Vespa（或 Qdrant multi-vector）配置、100 问多页评估集、viewer UI，以及一份说明文档，包含 content-class x approach 矩阵，并给出 2026 年哪些内容类别仍更适合 OCR-then-text 的具体建议。
