---
name: codebase-rag-zh
description: 构建一个跨仓库语义搜索系统，具备 AST 感知切块、混合检索、增量重建索引和带引用答案。
version: 1.0.0
phase: 19
lesson: 02
tags: [capstone, rag, code-search, tree-sitter, qdrant, bm25, hybrid-retrieval]
---

给定 10 个以上仓库，总计至少 200 万行代码，构建摄取流水线、混合索引，以及强制引用的查询智能体，让它能用可验证的 file:line 锚点回答跨仓库问题。

构建计划：

1. 用 tree-sitter 解析每个文件。在函数和类节点边界切块。存储 `{repo, path, start_line, end_line, symbol, body}`。
2. 用 Claude Haiku 4.5 或 Gemini 2.5 Flash 摘要每个分块，并对系统提示词使用 prompt caching。把一句话摘要和分块放在一起存储。
3. 索引到三种结构：Qdrant（稠密，Voyage-code-3 或 nomic-embed-code）、Tantivy（带字段权重的 BM25）和 kuzu（imports、calls、inheritance 的符号图边）。
4. 构建一个三节点 LangGraph 查询智能体：retrieve（dense 并行 BM25）、rerank（Cohere rerank-3 或 bge-reranker-v2-gemma-2b）、synth（Claude Sonnet 4.7，使用 prompt caching，并要求 file:line 引用）。
5. 后置过滤：拒绝任何没有可验证 `(repo/path:start-end)` 锚点的主张；重问或丢弃。
6. 接入 git push webhook，计算符号级 diff，只重新嵌入变更分块。目标：在 200 万 LOC 仓库群上，50 文件提交能在 60 秒内可搜索。
7. 用 100 问题留出集评估。报告 MRR@10、nDCG@10、引用忠实度和延迟百分位。
8. 运行每周漂移任务，重新执行评估，并在 MRR@10 下降超过 5% 时告警。

评估标准：

| Weight | Criterion | Measurement |
|:-:|---|---|
| 25 | 检索质量 | 在 100 问题留出集上的 MRR@10 和 nDCG@10 |
| 20 | 引用忠实度 | 答案主张中带有可验证 file:line 锚点的比例 |
| 20 | 延迟和规模 | 在已索引语料规模上，10k QPS 下的 p95 查询延迟 |
| 20 | 增量索引正确性 | 从 git push 到 50 文件提交可搜索所需时间 |
| 15 | UX 和答案格式 | 引用可点击性、代码片段预览、后续追问入口 |

硬性拒收：

- 用固定大小 token 切块代替 AST 感知切块。它会污染生成代码占比高的语料。
- 只有余弦检索，没有 BM25 或重排。已知它会在精确符号名查询上失败。
- 答案没有强制 file:line 引用。
- 每次 git push 都对全语料重新嵌入；必须是增量式。

拒绝规则：

- 拒绝在没有阅读许可证的情况下索引仓库。有些许可证禁止把代码嵌入到第三方向量存储。
- 拒绝回答声称引用了索引从未见过的文件的查询；返回前必须验证锚点。
- 拒绝在 p95 超过 4 秒时提供完整答案；改为返回部分结果和后续句柄。

输出：一个仓库，包含摄取流水线、LangGraph 查询智能体、100 问题标注评估集、Langfuse dashboard 链接，以及一份说明文档，列出你修复的三种检索失败模式（生成代码污染、长尾符号召回、跨仓库符号解析）和修复每个问题的具体变更。
