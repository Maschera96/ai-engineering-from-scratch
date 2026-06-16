# Capstone 02 — 代码库 RAG（跨仓库语义搜索）

> 到 2026 年，每个严肃的工程组织都会运行内部代码搜索，它理解的是语义，不只是字符串。Sourcegraph Amp、Cursor 的代码库问答、Augment 的企业图谱、Aider 的 repomap、Pinterest 的内部 MCP，形态都一样。摄取多个仓库，用 tree-sitter 解析，以函数和类为粒度切块并嵌入，混合搜索，重排，带引用回答。本综合实战要求你构建一个系统，它能处理 10 个仓库、200 万行代码，并且能在每次 git push 后完成增量重建索引。

**Type:** Capstone
**Languages:** Python（摄取）、TypeScript（API + UI）
**Prerequisites:** Phase 5（NLP 基础）、Phase 7（transformers）、Phase 11（LLM 工程）、Phase 13（工具）、Phase 17（基础设施）
**Phases exercised:** P5 · P7 · P11 · P13 · P17
**Time:** 30 小时

## 问题

到 2026 年，每个前沿编程智能体都会内置代码库检索层，因为仅靠上下文窗口无法解决跨仓库问题。Claude 的 100 万 token 上下文有帮助，但它并不能取消排序检索的必要性。对原始分块做朴素余弦搜索，会在生成代码、单体仓库重复代码，以及很少被导入的长尾符号上污染结果。生产级答案是混合搜索，也就是在 AST 感知分块上同时做稠密检索和 BM25，并加上重排器，背后由符号引用图支撑。

你会通过索引一组真实仓库来学习它，而不是只索引一个教程仓库。你要度量 MRR@10、引用忠实度和增量新鲜度。失败模式主要来自基础设施：10 万文件的单体仓库，一次触碰半数文件的 push，一个必须跨四个仓库才能正确回答的查询。

## 概念

AST 感知摄取流水线会用 tree-sitter 解析每个文件，抽取函数和类节点，并在节点边界切块，而不是按固定 token 窗口切块。每个分块有三种表示：稠密嵌入（Voyage-code-3 或 nomic-embed-code）、稀疏 BM25 词项，以及一段简短的自然语言摘要。摘要提供第三种可检索模态，用户问“X 是怎么授权的”，摘要里可能写着“authz”，哪怕代码里只有 `check_permission`。

检索是混合式的。一个查询会同时触发稠密搜索和 BM25 搜索，合并 top-k，再把并集交给 cross-encoder 重排器（Cohere rerank-3 或 bge-reranker-v2-gemma-2b）。重排后的列表进入长上下文综合器（带 prompt caching 的 Claude Sonnet 4.7，或自托管 Llama 3.3 70B），并要求每个主张都引用文件和行范围。没有引用的答案会被后置过滤器拒绝。

增量新鲜度才是基础设施难题。Git push 触发 diff：哪些文件变了，哪些符号变了。只有受影响的分块会重新嵌入。受影响的跨文件符号边（imports、method calls）会重新计算。索引保持一致，而不需要每次提交都重新处理 200 万行代码。

## 架构

```
git push --> webhook --> ingest worker (LlamaIndex Workflow)
                           |
                           v
              tree-sitter parse + AST chunk
                           |
            +--------------+----------------+
            v              v                v
          dense        BM25 index       summary (LLM)
        (Voyage / bge)  (Tantivy)        (Haiku 4.5)
            |              |                |
            +------> Qdrant / pgvector <----+
                            |
                            v
                      symbol graph (Neo4j / kuzu)
                            |
  query --> LangGraph agent (retrieve -> rerank -> synth)
                            |
                            v
                 Claude Sonnet 4.7 1M context
                            |
                            v
                 answer + file:line citations
```

## 技术栈

- 解析：tree-sitter，配 17 种语言语法（Python、TS、Rust、Go、Java、C++ 等）
- 稠密嵌入：Voyage-code-3（托管）或 nomic-embed-code-v1.5（自托管），bge-code-v1 作为后备
- 稀疏索引：Tantivy（Rust），使用 BM25F，对符号名和正文做字段加权
- 向量数据库：Qdrant 1.12，支持混合搜索；或 pgvector + pgvectorscale，适合低于 5000 万向量的团队
- 分块摘要模型：Claude Haiku 4.5 或 Gemini 2.5 Flash，使用 prompt caching
- 重排器：Cohere rerank-3 或自托管 bge-reranker-v2-gemma-2b
- 编排：LlamaIndex Workflows 用于摄取，LangGraph 用于查询智能体
- 综合器：Claude Sonnet 4.7（100 万上下文），使用 prompt caching
- 符号图：Neo4j（托管）或 kuzu（嵌入式），用于 import 边和 call 边
- 可观测性：每个检索和综合步骤记录 Langfuse spans

## 构建它

1. **摄取遍历器。** 在每个 push hook 上遍历 git 历史。收集变更文件。对每个文件用 tree-sitter 解析，抽取函数和类节点及其完整源码范围。输出分块记录 `{repo, path, start_line, end_line, symbol, body}`。

2. **分块摘要器。** 将分块批量送入 Haiku 4.5 调用，并对系统前言使用 prompt caching。提示词：“Summarize this function in one sentence, naming its public contract and side effects.” 将摘要和分块一起存储。

3. **嵌入池。** 两个并行队列：dense（Voyage-code-3 batch 128）和 summary（同一模型，但输入摘要字符串）。把向量写入 Qdrant，payload 为 `{repo, path, start_line, end_line, symbol, kind}`。

4. **BM25 索引。** 字段加权 Tantivy 索引：符号名权重 4，符号正文权重 1，摘要权重 2。这样既支持“查找名为 X 的函数”，也支持“查找做 X 的函数”。

5. **符号图。** 对每个分块记录边：imports（这个文件使用 repo Z 的 symbol Y）、calls（这个函数调用 class C 上的 method M）、inheritance。存入 kuzu。查询时用它把检索扩展到仓库边界之外。

6. **查询智能体。** LangGraph 有三个节点。`retrieve` 并行触发 dense + BM25，并按 (repo, path, symbol) 去重。`rerank` 在 top-50 上运行 cross-encoder，保留 top-10。`synth` 调用 Claude Sonnet 4.7，把重排后的分块放进上下文，缓存系统提示词，并要求 file:line 引用。

7. **引用强制。** 解析模型输出；任何没有 `(repo/path:start-end)` 锚点的主张，都标记为需要重问或直接丢弃。只把带引用的答案返回给用户。

8. **增量重建索引。** 每个 webhook 上计算符号级 diff。只有文本变化的分块会重新嵌入。只有 imports 变化的分块会重新计算符号边。目标度量：在 200 万 LOC 仓库群中，一次 50 文件提交能在 60 秒内完成重建索引。

9. **评估。** 标注 100 个跨仓库问题，并给出金标准 file:line 答案。度量 MRR@10、nDCG@10、引用忠实度（带可验证锚点的主张比例）以及 p50/p99 延迟。

## 使用它

```
$ code-rag ask "how is S3 multipart abort wired into our retry budget?"
[retrieve]  12 chunks dense + 7 chunks bm25, 16 unique after dedup
[rerank]    top-5 kept (cohere rerank-3)
[synth]     claude-sonnet-4.7, cache hit rate 68%, 2.1s
answer:
  Multipart aborts are triggered by `AbortMultipartOnFail` in
  services/uploader/retry.go:122-148, which decrements the per-bucket
  retry budget defined in config/budgets.yaml:34-51 ...
  citations: [services/uploader/retry.go:122-148, config/budgets.yaml:34-51,
              libs/s3client/multipart.ts:44-61]
```

## 交付它

可交付技能是 `outputs/skill-codebase-rag.md`。给定一组仓库语料，它会启动摄取流水线、混合索引和查询智能体，并为任何跨仓库问题返回带引用的答案。评分标准：

| Weight | Criterion | How it is measured |
|:-:|---|---|
| 25 | 检索质量 | 在 100 问题留出集上的 MRR@10 和 nDCG@10 |
| 20 | 引用忠实度 | 答案主张中带有可验证 file:line 锚点的比例 |
| 20 | 延迟和规模 | 在已索引语料规模上，10k QPS 下的 p95 查询延迟 |
| 20 | 增量索引正确性 | 从 git push 到 50 文件提交可搜索所需时间 |
| 15 | UX 和答案格式 | 引用可点击性、代码片段预览、后续追问入口 |
| **100** | | |

## 练习

1. 将 Voyage-code-3 替换为自托管 nomic-embed-code。度量 MRR@10 变化。报告启用重排后差距是否缩小。

2. 向语料注入 20% 生成代码（LLM 生成的样板代码），然后重新评估。观察检索污染。给 payload 增加 `generated` 标记，并降低这些命中的权重。

3. 在你的语料规模上对比 Qdrant hybrid search 与 pgvector + pgvectorscale。报告 batch size 1 下的 p99。

4. 添加基于抽样的漂移检查：每周重新运行 100 问题评估。MRR@10 下降超过 5% 时告警。

5. 扩展到跨语言符号解析：一个 Python 函数通过 gRPC 调用 Go 服务。用符号图把它们连接起来。

## 关键术语

| Term | What people say | What it actually means |
|------|-----------------|------------------------|
| AST-aware chunking | “函数级切分” | 在 tree-sitter 节点边界切分代码，而不是按固定 token 窗口切分 |
| Hybrid search | “稠密 + 稀疏” | 并行运行 BM25 和向量搜索，合并 top-k，再重排 |
| Cross-encoder rerank | “第二阶段排序” | 把每个 (query, candidate) 对一起打分的模型，比余弦相似度更准确 |
| Prompt caching | “缓存系统提示词” | 2026 年 Claude / OpenAI 功能，可让重复前缀 token 最多降价 90% |
| Symbol graph | “代码图” | 跨文件和跨仓库记录 imports、calls、inheritance 的边 |
| Citation faithfulness | “有根据答案率” | 用户能点击锚点并阅读引用片段来验证的主张比例 |
| Incremental re-index | “push-to-search 时间” | 从 git push 到变更符号可被查询的墙钟时间 |

## 延伸阅读

- [Sourcegraph Amp](https://ampcode.com) — 生产级跨仓库代码智能
- [Sourcegraph Cody RAG architecture](https://sourcegraph.com/blog/how-cody-understands-your-codebase) — 本综合实战的参考深度解析
- [Aider repo-map](https://aider.chat/docs/repomap.html) — tree-sitter 排序仓库视图
- [Augment Code enterprise graph](https://www.augmentcode.com) — 商业符号图 RAG
- [Qdrant hybrid search docs](https://qdrant.tech/documentation/concepts/hybrid-queries/) — 参考实现
- [Voyage AI code embeddings](https://docs.voyageai.com/docs/embeddings) — Voyage-code-3 细节
- [Cohere rerank-3](https://docs.cohere.com/reference/rerank) — cross-encoder 参考
- [Pinterest MCP internal search](https://medium.com/pinterest-engineering) — 内部平台参考
