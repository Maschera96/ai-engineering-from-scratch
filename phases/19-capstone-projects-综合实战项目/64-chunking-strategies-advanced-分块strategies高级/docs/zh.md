# 分块策略对比

> 分块决定了检索器最终能返回什么。边界切错后，任何嵌入模型、重排器或 LLM 都无法在下游修复损害。

**Type:** Build
**Languages:** Python
**Prerequisites:** Phase 11 lessons 04 (embeddings), 06 (RAG), 07 (advanced RAG); Phase 19 Track B foundations (lessons 20-29)
**Time:** ~90 minutes

## Learning Objectives
- 从零实现五种分块策略：fixed-window、sentence、recursive-split、semantic clustering 和 structural markdown headers。
- 在带有 gold-labeled answer spans 的 fixture 语料上测量 recall@k，并解释为什么一种策略在散文上胜出，另一种策略在技术文档上胜出。
- 读取块长度分布，并识别每种策略注入的失败模式：孤立句、符号中间切断、只有标题的块、语义漂移。
- 不运行基准，也能通过检查三项属性为新语料选择默认策略：文档类型、平均段落长度，以及格式是否带显式结构。

## 问题

每个 RAG 流水线都从把源文档切成片段开始。片段要足够小，嵌入模型才能容纳；也要足够大，每个片段才能承载一个自包含想法。切在哪里不是普通超参数。它是检索器最终能返回什么的上限。

一个询问 “what does the budget abort threshold look like” 的查询，只有在包含 abort threshold 的块可以被检索到时才可能成功。如果 fixed-window splitter 把阈值从周围上下文中切开，嵌入会移到不同聚类，BM25 分数下降，重排器看到噪声，LLM 生成的答案就会出错。2024 年论文 “LongRAG: Enhancing Retrieval-Augmented Generation with Long-context LLMs” 测得，仅分块选择就能让检索召回产生 35 个百分点的绝对差异。2025 年关于 contextual chunk headers 的后续工作缩小了差距，但没有消除它。

本课会并排构建五种策略，在带 gold-labeled answer spans 的 fixture 语料上运行，并让你自己阅读召回数字。

## 概念

```mermaid
flowchart LR
  Doc[源文档] --> S1[固定窗口]
  Doc --> S2[句子]
  Doc --> S3[递归切分]
  Doc --> S4[语义聚类]
  Doc --> S5[结构化 Markdown]
  S1 --> Chunks1[块]
  S2 --> Chunks2[块]
  S3 --> Chunks3[块]
  S4 --> Chunks4[块]
  S5 --> Chunks5[块]
  Chunks1 --> Index[嵌入索引]
  Chunks2 --> Index
  Chunks3 --> Index
  Chunks4 --> Index
  Chunks5 --> Index
  Index --> Eval[Recall@k vs Gold Spans]
```

### Fixed-window

暴力基线。每 N 个字符切一次。可以选择重叠，让在位置 N 被切断的句子完整出现在从位置 N 减 overlap 开始的块里。快速、确定、边界很糟。把它当作对照，不要当默认值。

### Sentence

用正则或简单状态机按句子边界切分。把一个或多个句子打包进一个块，直到达到目标字符预算。不再切断单词。仍然会切断段落和章节。它是很多早期 RAG 流水线的默认值，对没有其他结构的散文是合理选择。

### Recursive split

这是 2023 年代库普及的层次策略。先尝试最强分隔符，双换行、段落；再回退到下一个，单换行；然后句子；最后字符。当块适配预算时递归结束。对于结构不一致的文档很强，因为它能按区域自适应。

### Semantic clustering

嵌入每个句子。把共享主题质心的连续句子聚成一组。当运行相似度与质心低于阈值时切开。边界反映含义，而不是字符。构建更慢且依赖嵌入模型，但能抵抗在一个段落内切换主题的文档。

### Structural markdown headers

对于带显式结构的文档，markdown、reStructuredText、RFC 风格编号章节，在标题边界切分。每个块包含标题以及其下方直到同级或更高级标题前的所有内容。每个主题的块最小，但只有在语料格式良好时可用。

### recall@k 如何测量边界选择

gold-labeled 查询带有答案片段在源文档中的精确字符偏移。分块之后，你要问：检索器返回的 top-k 块中，是否有任何一个与 gold span 重叠？如果有，该查询的 recall@k 为 1。否则为 0。对查询集取平均。对每种策略运行同一评估，差距会显示哪种边界策略适合你手头的语料。

## 构建

`code/main.py` 实现：

- `fixed_window(text, size, overlap)`，基线。
- `sentence_chunks(text, target)`，简单句子打包器。
- `recursive_split(text, separators, target)`，层次递归。
- `semantic_chunks(text, similarity_threshold)`，基于确定性模拟嵌入的质心聚类。
- `structural_markdown(text)`，感知标题的切分器。
- `mock_embed(text, dim)`，基于哈希的嵌入，让循环离线运行。
- `DenseIndex`，与 Phase 19 Track B 混合检索课使用的形状相同。
- `eval_recall(strategy, corpus, queries, k)`，比较循环。
- 一个 `main()`，在 fixture 语料上运行每种策略，并打印 recall@k 表。

运行：

```bash
python3 code/main.py
```

输出是一个小表，每个策略一行，每个 k 一列。Sentence 在结构化 fixture 上失利。Structural-markdown 在 markdown fixture 上获胜。Recursive 在混合 fixture 上表现不错，因为递归能自适应。Semantic clustering 在没有有用结构线索的散文 fixture 上获胜。

## 表格不会隐藏的失败模式

**孤立句。** Sentence packing 会产生缺少主题句的块。嵌入随后指向错误聚类。

**符号中间切断。** Fixed-window 在代码或 YAML 中会把标识符切成两半。两半都会嵌入成噪声。

**只有标题的块。** Structural markdown 会输出只包含 `## Title` 的块。过滤掉它们，或附上后一个块的第一段。

**语义漂移。** 当语料整体主题统一时，semantic clustering 会切得不够。一个 5000 字符块会把很多具体答案装进一个发散嵌入里。把 semantic 与硬字符上限组合起来。

**过期嵌入。** Semantic clustering 使用嵌入模型。如果更换模型，也就更换了分块。单独固定分块模型，或与检索模型一起重建索引。

## 不运行基准也能选择默认值

三个属性决定新语料的默认分块器。

| Property | Value | Default |
|----------|-------|---------|
| Document type | 无结构散文 | Recursive split, target 800 |
| Document type | Markdown / RFC / API docs | Structural markdown |
| Document type | Code | AST-aware，超出本课范围，见 Phase 19 lesson 02 |
| Paragraph length | 长，单一主题 | Sentence, target 500 |
| Paragraph length | 短，主题混杂 | Semantic, threshold 0.6 |

拿不准时，选 recursive split。它是最强的单策略基线。

## 使用

生产模式：

- 在发布新流水线前运行评估，不要相信库的默认策略。
- 每当更换嵌入模型或语料组合时重新运行评估，赢家依赖语料。
- 在每个块的 metadata 中持久化策略名称，这样之后可以归因回归。

## 交付

第 69 课的 Track F 端到端 RAG 系统会把这里选出的分块器作为第一阶段。第 68 课的评估测试框架读取的 recall@k 与本课 `eval_recall` 返回的形状相同。选出在你语料上胜出的策略，并把它传递下去。

## 练习

1. 添加第六种策略：使用 `tiktoken` 而非字符计数的 token-window。和同一 fixture 上的 fixed-window 比较。
2. 向散文 fixture 注入 30% 的代码块。重新运行表格。解释为什么除了 structural markdown 之外的每种策略都会损失召回。
3. 用项目真实 provider 的嵌入替换确定性嵌入。测量 semantic-clustering 的召回变化。报告策略之间的差距是扩大还是缩小。
4. 为每个块添加 `summary` 字段：一句话质心描述。把 summary 附加到块正文后重新运行评估。测量召回提升。

## 关键术语

| Term | What people say | What it actually means |
|------|-----------------|------------------------|
| Recall@k | “Did we get the right chunk?” | top-k 块中任意块与 gold answer span 重叠的查询比例 |
| Chunk overlap | “Sliding window” | 把前一个块的最后 N 个字符重新包含在下一个块中 |
| Structural splitter | “Header-aware chunks” | 在 H1/H2/H3 边界切分，标题文本是块的一部分 |
| Semantic chunker | “Topic-aware chunks” | 嵌入句子，按质心相似度聚类，在漂移处切分 |
| Centroid drift | “Topic shift” | 运行均值和下一句之间的余弦相似度低于阈值 |

## 延伸阅读

- [LongRAG: Enhancing Retrieval-Augmented Generation with Long-context LLMs (arXiv 2406.15319)](https://arxiv.org/abs/2406.15319)
- [Anthropic, Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval)
- [LlamaIndex, Chunking strategies for production RAG](https://docs.llamaindex.ai/en/stable/optimizing/production_rag/)
- Phase 11 lesson 06，RAG fundamentals
- Phase 11 lesson 07，advanced RAG
- Phase 19 lesson 65，索引这里产生的块的混合检索
- Phase 19 lesson 68，在生产中评估策略选择的测试框架
