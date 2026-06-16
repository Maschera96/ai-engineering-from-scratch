# Capstone 08 — 用于受监管行业的生产 RAG 聊天机器人
> Harvey、Glean、Mendable 和 LlamaCloud 都将在 2026 年运行相同的生产形态。使用 dobling 或 Unstructed 和 ColPali 进行摄取以获取视觉效果。混合搜索。使用 bge-reranker-v2-gemma 重新排名。使用提示缓存以 60-80% 的命中率与 Claude Sonnet 4.7 进行合成。使用 Llama Guard 4 和 NeMo 护栏进行防护。使用 Langfuse 和 Phoenix 观看。使用 RAGAS 对 200 道黄金题进行评分。在受监管的领域（法律、临床、保险）建立一个，顶峰是通过黄金组、红队和漂移仪表板。
**类型：** 顶点
**语言：** Python（管道 + API）、TypeScript（聊天 UI）
**先决条件：** 第 5 阶段 (NLP)、第 7 阶段（变压器）、第 11 阶段（LLM 工程）、第 12 阶段（多式联运）、第 17 阶段（基础设施）、第 18 阶段（安全）
**练习阶段：** P5 · P7 · P11 · P12 · P17 · P18
**时间：** 30小时
＃＃ 问题
受监管领域RAG（法律合同、临床试验协议、保险单）是 2026 年出货量最大的生产形式，因为投资回报率是显而易见的，而且风险是具体的。 Harvey (Allen & Overy) 建造它是为了合法。 Mendable 具有开发人员文档风格。 Glean 涵盖企业搜索。该模式是：摄取高保真度、通过重新排序检索混合体、通过引用强制和提示缓存进行合成、使用多个安全层进行防护，并持续监控漂移。
困难的部分不是模型。它们是司法管辖区合规性（HIPAA、GDPR、SOC2）、引文级别可审核性、成本控制（命中率高时提示缓存可享受 60-90% 的折扣）、通过 RAGAS 忠实度进行幻觉检测，以及在源文档更新而索引未跟上时进行偏差检测。这个顶点要求您将所有内容放在包含 200 个问题的黄金套装中，并附上红队套件。
＃＃ 概念
管道有两侧。 **摄取**：docling或Unstructed解析结构化文档； ColPali 处理视觉丰富的内容；块获取摘要、标签和基于角色的访问标签。矢量进入 pgvector + pgvectorscale（50M 以下矢量）或 Qdrant Cloud；稀疏的 BM25 并排运行。 **对话**：LangGraph处理内存和多回合；每个查询运行混合检索，使用 bge-reranker-v2-gemma-2b 重新排序，使用 Claude Sonnet 4.7（提示缓存）进行合成，通过 Llama Guard 4 和 NeMo Guardrails 传递输出，并发出引文锚定响应。
eval 堆栈有四层。 **黄金套装**（200 个标记为 Q/A 并带有引文）以确保正确性。 **红队**（越狱、PII 提取尝试、域外问题）以确保安全。 **RAGAS** 每回合自动实现忠实度/答案相关性/上下文精确度。 **漂移仪表板** (Arize Phoenix) 每周观察检索质量和幻觉得分。
即时缓存是成本杠杆。 Claude 4.5+ 和 GPT-5+ 支持缓存系统提示 + 检索的上下文。当命中率达到 60-80% 时，每次查询成本会下降 3-5 倍。管道必须针对稳定的前缀（系统提示+首先重新排序的上下文）进行设计，以实现高缓存命中率。
＃＃ 建筑学
```
documents (contracts, protocols, policies)
      |
      v
docling / Unstructured parse + ColPali for visuals
      |
      v
chunks + summaries + role-labels + jurisdiction tags
      |
      v
pgvector + pgvectorscale  +  BM25 (Tantivy)
      |
query + role + jurisdiction
      |
      v
LangGraph conversational agent
   +--- retrieve (hybrid)
   +--- filter by role + jurisdiction
   +--- rerank (bge-reranker-v2-gemma-2b or Voyage rerank-2)
   +--- synthesize (Claude Sonnet 4.7, prompt cached)
   +--- guard (Llama Guard 4 + NeMo Guardrails + Presidio output PII scrub)
   +--- cite + return
      |
      v
eval:
  RAGAS faithfulness / answer_relevance / context_precision (online)
  Langfuse annotation queue (sampled)
  Arize Phoenix drift (weekly)
  red team suite (pre-release)
```

＃＃ 堆
- 摄取：Unstructured.io 或结构化文档的文档； ColPali 提供视觉效果丰富的 PDF
- 矢量DB：50M矢量以下的pgvector + pgvectorscale； Qdrant 云否则
- 稀疏：带有字段权重的 Tantivy BM25
- 编排：LlamaIndex 工作流程（摄取）+ LangGraph（对话）
- 重新排名器：bge-reranker-v2-gemma-2b 自托管或 Voyage rerank-2 托管
- LLM：Claude Sonnet 4.7，带提示缓存；后备 Llama 3.3 70B 自托管
- 评估：RAGAS 0.2在线，DeepEval用于幻觉和越狱套件
- 可观察性：Langfuse 自托管，带有注释队列； Arize Phoenix 进行漂移
- Guardrails：Llama Guard 4 input/output 分类器、NeMo Guardrails v0.12 策略、Presidio PII 清理
- 合规性：块上基于角色的访问标签； GDPR/HIPAA 的管辖权标签
```figure
canary-rollout
```

## 构建它
1. **摄取。** 使用非结构化或文档解析您的语料库（认真构建 1000-10000 个文档）。对于扫描/视觉密集的页面，请通过 ColPali 进行路由。生成带有摘要、角色标签、管辖权标签的块。
2. **Index.** 密集嵌入（Voyage-3 或 Nomic-embed-v2）到 pgvector + pgvectorscale 中。 BM25 通过 Tantivy 的辅助索引。角色和管辖权过滤器作为有效负载。
3. **混合检索。** 首先按角色+管辖权过滤；然后并行密集 + BM25;与倒数等级融合合并；重新排名前 20 名；合成前 5 名。
4. **与提示缓存合成。**系统提示+缓存头中的静态策略；将上下文重新排序为缓存扩展；用户问题作为未缓存的后缀。稳定状态下缓存命中率目标为 60-80%。
5. **Guardrails。** Llama Guard 输入 4； NeMo Guardrails 可以阻止域外问题或政策禁止的主题； Presidio 清除输出中的意外 PII；引文执行后过滤器。
6. **黄金套装。** 200 个 Q/A 对，由领域专家标记为（答案、引文）。对代理的精确引文匹配、答案正确性、忠实度进行评分 (RAGAS)。
7. **红队。** 50 个对抗性提示：越狱（PAIR、TAP）、PII 渗透尝试、域外、跨管辖区泄漏。根据 pass/fail 和严重性进行评分。
8. **漂移仪表板。** Arize Phoenix 每周跟踪检索质量（nDCG，引文忠实度）。下跌 5% 时发出警报。
9. **成本报告。** Langfuse：提示缓存命中率、每个查询的令牌、按阶段的 $/查询细分。
## 使用它
```
$ chat --role=analyst --jurisdiction=GDPR
> what is the data-retention obligation for EU user profiles under our contract?
[retrieve]  hybrid top-20 filtered to GDPR + analyst-role
[rerank]    top-5 kept
[synth]     claude-sonnet-4.7, cache hit 74%, 0.8s
answer:
  The contract (Section 12.4, Master Services Agreement dated 2024-03-11)
  obligates EU user profile deletion within 30 days of termination per GDPR
  Article 17. The DPA amendment (DPA-v2.1, Section 5) extends this to 14 days
  for "restricted" category data.
  citations: [MSA-2024-03-11 s12.4, DPA-v2.1 s5]
```

## 发货
<span class="notranslate">`outputs/skill-production-rag.md`</span> 描述了可交付成果。一个受监管的域聊天机器人部署了合规性标签，通过了规则，通过实时漂移监控进行观察。
|重量 |标准|如何测量 |
|:-:|---|---|
| 25 | 25 RAGAS 忠诚度 + 答案相关性 |黄金组在线分数 (200 Q/A) |
| 20 |引用正确性 |具有可验证源锚点的答案比例 |
| 20 |护栏覆盖范围| Llama Guard 4 通过率 + 越狱套件结果 |
| 20 |成本/延迟工程|提示缓存命中率、p95 延迟、$/query |
| 15 | 15漂移监控仪表板| Phoenix 实时仪表板，包含每周检索质量趋势 |
| **100** | | |
## 练习
1. 在不同管辖范围内构建第二个语料库切片（例如，HIPAA 和 GDPR）。演示角色+管辖区过滤，防止 20 个问题的跨管辖区调查出现交叉泄露。
2. 测量一周生产流量的提示缓存命中率。确定哪些查询破坏了缓存前缀。重组。
3. 添加具有 10k-token 摘要缓冲区的多轮内存。衡量忠诚度是否会随着对话的进行而下降。
4. 将 Claude Sonnet 4.7 替换为 Llama 3.3 70B 自托管。衡量 $/query 和忠诚度增量。
5. 添加“不确定”模式：如果重新排序的最高分数低于阈值，代理会说“我没有自信的引用”而不是回答。衡量错误置信度的减少。
## 关键术语
|术语 |人们怎么说|它实际上意味着什么 ||------|-----------------|------------------------|
|提示缓存 | “缓存系统+上下文” | Claude/OpenAI 功能：缓存的前缀标记命中折扣 60-90% |
|拉格斯 | “RAG 评估者”|自动评分忠实度、答案相关性、上下文精确度 |
|金色套装| “标记评估”| 200 多个专家标记的 Q/A 以及引用；事实真相|
|司法管辖区标签 | “合规标签” | GDPR/HIPAA/SOC2 附加到块的范围；由检索过滤器强制 |
|引文忠实度 | “接地答复率” |由可检索来源跨度支持的声明比例|
|漂移| “检索质量下降” | nDCG 或引文分数每周发生变化；警报阈值 5% |
|红队| “对抗性评估” |预发布越狱、PII 提取、域外探测 |
## 进一步阅读
- [Harvey AI](<span class="notranslate">https://www.harvey.ai</span>) — 参考合法生产堆栈
- [收集企业搜索](<span class="notranslate">https://www.glean.com</span>) — 参考企业规模的RAG
- [Mendable 文档](<span class="notranslate">https://mendable.ai</span>) — 开发人员文档 RAG 参考
- [LlamaCloud Parse + Index](<span class="notranslate">https://docs.llamaindex.ai/en/stable/examples/llama_cloud/llama_parse/</span>) — 托管摄取
- [Anthropic 提示缓存](<span class="notranslate">https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching</span>) — 成本杠杆参考
- [RAGAS 0.2 文档](<span class="notranslate">https://docs.ragas.io/</span>) — 规范的 RAG 评估框架
- [Arize Phoenix](<span class="notranslate">https://github.com/Arize-ai/phoenix</span>) — 参考漂移可观测性
- [Llama Guard 4](<span class="notranslate">https://ai.meta.com/research/publications/llama-guard-4/</span>) — 2026 年安全分类器
- [NeMo Guardrails v0.12](<span class="notranslate">https://docs.nvidia.com/nemo-guardrails/</span>) — 政策轨道框架