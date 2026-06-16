---
name: vision-rag-designer-zh
description: 使用 ColPali / ColQwen2 / VisRAG 设计视觉原生文档 RAG，附带存储估算与生成器选型。
version: 1.0.0
phase: 12
lesson: 23
tags: [colpali, colqwen2, visrag, late-interaction, vidore]
---

给定一个文档 RAG 项目（语料规模、查询延迟目标、存储预算、单次查询成本），输出一份视觉原生 RAG 配置。

请产出：

1. 检索器选型。ColPali（PaliGemma 基座）、ColQwen2（Qwen2-VL 基座，质量更好）、ColSmol（1B，适合边缘端）或 VisRAG（双编码器，存储更便宜）。
2. 存储估算。原始大小为 N_docs * N_p_per_doc * D * 4 bytes；采用 PQ 时除以 8。
3. 延迟估算。
   - 检索 SLA：约 10ms 查询嵌入 + top-k 检索（MaxSim 或 ANN），取决于索引大小。
   - 完整回答 SLA：检索延迟 + 200-500ms 生成器（取决于模型和硬件）。
4. 生成器选型。开源选 Qwen2.5-VL-72B，前沿选 Claude Opus 4.7。
5. 压缩方案。PQ / OPQ 比率目标 8-16 倍；用 HNSW 索引实现快速 ANN。
6. 从 text-RAG 的迁移路径。如何 A/B、何时完全切换。

硬性拒绝项：
- 在 >10k 页的语料上不使用 PQ 压缩就使用 ColPali。存储会爆炸。
- 声称双编码器检索在文档召回上能匹敌 ColBERT MaxSim。在 ViDoRe 上它做不到。
- 为图表 + 表格工作负载推荐 text-RAG。text-RAG 会丢失大部分信号。

拒绝规则：
- 如果语料是纯文本（wiki、聊天记录），拒绝视觉原生 RAG，并推荐标准 text-RAG。
- 如果检索 SLA <100ms，优先选 VisRAG（双编码器）而非 ColPali MaxSim。
- 如果完整回答 SLA <100ms，完全拒绝生成式 RAG，并推荐仅检索的 UX 或缓存答案。
- 如果存储预算 <1 GB 且语料 >100k 页，拒绝全保真 ColPali；提议激进的 PQ 或 VisRAG。

输出：一页纸的 RAG 设计，包含检索器选型、存储估算、延迟、生成器、压缩、迁移。结尾附上 arXiv 2407.01449（ColPali）、2410.10594（VisRAG）。
