# ColPali 与视觉原生文档 RAG

> 传统 RAG 把 PDF 解析为文本，切分为分块，对分块做嵌入，再存储向量。每一步都会丢失信号：OCR 丢掉图表数据，分块打断表格行，文本嵌入忽略图形。ColPali（Faysse 等，2024 年 7 月）提出了一个更简单的问题：为什么要提取文本？直接通过 PaliGemma 对页面图像做嵌入，用 ColBERT 风格的延迟交互（late interaction）做检索，保留文档携带的全部布局、图形、字体和格式信号。已发表的基准测试结果：在视觉信息丰富的文档上，端到端准确率比文本 RAG 高 20-40%。ColQwen2、ColSmol 和 VisRAG 进一步扩展了这一范式。本课会研读视觉原生 RAG 的核心主张，并构建一个微型的类 ColPali 索引器。

**类型：** 构建
**语言：** Python（标准库，多向量索引器 + MaxSim 评分器）
**前置知识：** 第 11 阶段（LLM 工程 —— RAG 基础）、第 12 阶段 · 05（LLaVA）
**时长：** 约 180 分钟

## 学习目标

- 解释双编码器检索（每个文档一个向量）与延迟交互检索（每个文档多个向量）之间的区别。
- 描述 ColBERT 的 MaxSim 操作，以及 ColPali 如何将它从文本 token 推广到图像 patch。
- 构建一个微型的类 ColPali 索引器：页面 → patch 嵌入 → 对查询词嵌入做 MaxSim → 取 top-k 页面。
- 在发票 / 财务报告这类用例上，比较 ColPali + Qwen2.5-VL 生成器与文本 RAG + GPT-4。

## 问题所在

PDF 上的文本 RAG 丢弃了文档的大部分内容。财务报告的 Q3 营收增长通常在图表里；医疗报告的发现藏在带标注的图像中；法律合同的签名栏是一种布局事实，而非文本事实。

文本 RAG 流水线：

1. PDF → 通过 OCR / pdftotext 转为文本。
2. 文本 → 300-500 token 的分块。
3. 分块 → 双编码器嵌入（一个向量）。
4. 用户查询 → 嵌入 → 余弦相似度 → top-k 分块。
5. 分块 + 查询 → LLM。

五个有损步骤。图表未被捕获。表格被跨分块割裂。多栏布局被压平。图形标注消失。

ColPali 的解法：跳过 OCR，直接对页面图像做嵌入。用 ColBERT 风格的延迟交互做检索，让模型在查询时能关注细粒度的 patch。

## 核心概念

### ColBERT（2020）

ColBERT（Khattab & Zaharia，arXiv:2004.12832）是一种文本检索方法。它不是每个文档一个向量，而是每个 token 一个向量。查询时：

- 查询 token 获得各自的嵌入（N_q 个向量）。
- 文档 token 获得嵌入（N_d 个向量，通常会缓存）。
- 得分 = 对查询 token 求和，对每个查询 token 取文档 token 余弦相似度的最大值：Σ_i max_j cos(q_i, d_j)。

这就是 MaxSim 操作。每个查询 token “挑选”与它最匹配的文档 token。最终得分是这些值之和。

优点：召回强，能处理词级语义。缺点：每个文档 N_d 个向量，存储开销大。

### ColPali

ColPali（Faysse 等，arXiv:2407.01449）把 ColBERT 范式应用到图像上。

- 每个页面由 PaliGemma（ViT + 语言）编码为 patch 嵌入：每页 N_p 个向量。
- 每个用户查询（文本）被编码为查询 token 嵌入：N_q 个向量。
- 得分 = Σ_i max_j cos(q_i, p_j)，即对查询文本 token 与页面图像 patch 做 MaxSim。
- 按总分检索 top-k 页面。

文档摄取阶段：用 PaliGemma 对每页做嵌入，存储所有 patch 嵌入。查询阶段：对查询 token 做嵌入，与所有已存储的页面嵌入计算 MaxSim，返回 top-k 页面。

优点：在视觉信息丰富的文档上，端到端比文本 RAG 高 20-40%。每个 patch 向量都捕获局部布局与内容。

缺点：每页 N_p 个 patch × 4 字节浮点数 × D 维向量 = 存储增长很快。可用 PQ / OPQ 量化来缓解。

### ColQwen2 与 ColSmol

ColQwen2（illuin-tech，2024-2025）把 PaliGemma 换成 Qwen2-VL。更好的基础编码器，更好的检索。

ColSmol 是面向本地 / 边缘使用的更小规模变体。一个约 1B 参数的 ColSmol 检索器能在消费级 GPU 上运行。

### VisRAG

VisRAG（Yu 等，arXiv:2410.10594）是一个不同的变体：它不是在 patch 上做 MaxSim，而是用一个 VLM 把每页池化为单个向量，再做双编码器检索。索引更快、存储更小，但召回较弱。

质量与成本的权衡：要质量用 ColPali，要规模用 VisRAG。

### M3DocRAG

M3DocRAG（Cho 等，arXiv:2411.04952）把多模态检索扩展到多页、多文档推理。它跨文档检索页面，为 VLM 组合一个多页上下文。

### ViDoRe —— 基准

ColPali 的配套基准。Visual Document Retrieval Evaluation（视觉文档检索评估）。任务涵盖财务报告、科学论文、行政文档、医疗记录、操作手册。指标：nDCG@5。

ColPali-v1 在 ViDoRe 上达到约 80% 的 nDCG@5；同样文档上的文本 RAG 只有约 50-60%。

### 端到端 RAG 流水线

对于视觉原生 RAG：

1. 摄取：PDF → 页面图像 → PaliGemma 编码 → 存储所有 patch 嵌入。
2. 查询：用户文本 → 查询 token 嵌入 → 对所有已索引页面做 MaxSim → top-k 页面。
3. 生成：top-k 页面图像 + 查询 → VLM（Qwen2.5-VL 或 Claude）→ 答案。

全程无 OCR。图形、图表、字体、布局都流入答案。

### 存储算术

一份 50 页的财务报告，每页 729 个 patch，128 维嵌入：

- ColPali：50 * 729 * 128 * 4 字节 = 约 18 MB 原始，PQ 后约 4 MB。
- 文本 RAG：50 个分块 * 768 维 * 4 字节 = 约 150 kB。

ColPali 每份文档的存储约是其 30 倍。规模化时，OPQ / PQ 能把它降到约 5-10 倍，通常可以接受。

### 文本 RAG 仍然胜出的场景

- 没有布局信号的纯文本文档（维基文章、聊天记录）。文本 RAG 更简单，存储更省。
- 存储主导成本的数百万页级档案。
- 严格的合规要求，需要在检索之外提供可提取的 OCR 文本。

而在 2026 年的其他几乎所有场景里 —— 财务报告、科学论文、法律合同、医疗记录、UX 文档 —— 视觉原生 RAG 胜出。

## 动手用

`code/main.py`：

- 玩具 patch 编码器：把一个“页面”（特征向量的小网格）映射为一组 patch 嵌入。
- MaxSim 评分器：计算一组查询 token 嵌入与一个页面 patch 集之间的 ColBERT 风格得分。
- 索引 5 个玩具页面，运行 3 个查询，返回带得分的 top-k。

## 交付

本课产出 `outputs/skill-vision-rag-designer.md`。给定一个文档 RAG 项目，选择 ColPali / ColQwen2 / VisRAG / 文本 RAG 并估算存储规模。

## 练习

1. 一份 200 页的年报，每页 729 个 patch、128 维嵌入、4 字节浮点数。计算原始存储和 PQ 压缩（8 倍）后的存储。

2. MaxSim 是 Σ_i max_j cos(q_i, p_j)。相比简单的平均相似度，这个求和捕获了什么？

3. ColPali 把页面索引为 patch 集。如果改为在词级别索引（像 ColBERT 那样），会有什么变化？有哪些权衡？

4. 为一个 100 万页的语料库设计端到端流水线，每次查询的延迟预算为 500ms。选择 ColQwen2 / VisRAG 并说明理由。

5. 阅读 M3DocRAG（arXiv:2411.04952）。描述其多页注意力模式，以及它与单页 ColPali 检索的区别。

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|-----------------|------------------------|
| 延迟交互（Late interaction） | “ColBERT 风格” | 使用每个 token 或每个 patch 的嵌入 + MaxSim 进行检索，而非单个文档向量 |
| MaxSim | “对 patch 取最大” | 对每个查询 token，挑选相似度最高的文档 token；再对查询求和 |
| 双编码器（Bi-encoder） | “单向量” | 每个文档一个向量；更快但损失粒度 |
| 多向量（Multi-vector） | “每文档多向量” | 每个文档 / 页面存储 N_p 个向量；存储成本增长但召回提升 |
| Patch 嵌入 | “页面特征” | 来自 VLM 编码器的每个图像 patch 一个向量，按页缓存 |
| ViDoRe | “视觉文档基准” | ColPali 用于视觉文档检索的基准套件 |
| PQ 量化 | “乘积量化” | 在保持向量相似度的同时把存储缩小约 8 倍的压缩方法 |

## 延伸阅读

- [Faysse 等 —— ColPali (arXiv:2407.01449)](https://arxiv.org/abs/2407.01449)
- [Khattab & Zaharia —— ColBERT (arXiv:2004.12832)](https://arxiv.org/abs/2004.12832)
- [Yu 等 —— VisRAG (arXiv:2410.10594)](https://arxiv.org/abs/2410.10594)
- [Cho 等 —— M3DocRAG (arXiv:2411.04952)](https://arxiv.org/abs/2411.04952)
- [illuin-tech/colpali GitHub](https://github.com/illuin-tech/colpali)
