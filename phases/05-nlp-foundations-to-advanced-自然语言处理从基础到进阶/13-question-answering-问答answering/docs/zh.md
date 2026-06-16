# 问答系统

> 说明：Three systems shaped 现代 QA. Extractive found spans. Retrieval-augmented grounded them in 文档. Generative produced answers. Every 现代 AI assistant is a mix of the three.

**类型:** 构建
**语言:** Python
**先修要求:** 说明：Phase 5 · 11 (Machine Translation), Phase 5 · 10 (注意力 Mechanism)
**时间:** 约75分钟

## 问题

一个 用户 types "When did the first iPhone launch?" 与 expects "June 29, 2007." Not "Apple's 历史 is 长 与 varied." Not "2007" sitting in isolation 使用 no 句子. A direct, grounded, 正确 答案.

说明：Three architectures have dominated QA over the last decade.

- **Extractive QA.** 给定一个 问题 与 a passage 这 is known to contain the 答案, find the start 与 end indices of the 答案 span in the passage. SQuAD is the canonical 基准.
- **开放-领域 QA.** The passage is not given. Retrieve the relevant passage first, then extract 或 generate an 答案. This is the bedrock of every RAG 流水线 today.
- **Generative / 封闭-book QA.** A 大 语言模型 answers 从 its parametric memory. No 检索. Fastest at 推理, least reliable on facts.

这个 trend in 2026 is hybrid: retrieve the best few passages, then prompt a generative 模型 to 答案 grounded in those passages. 这 is RAG, 与 lesson 14 covers the 检索 half in depth. This lesson builds the QA half.

## 概念

说明：![说明：QA architectures: extractive, 检索-augmented, generative](../assets/qa.svg)

**Extractive.** Encode 问题 与 passage together 使用 a transformer (BERT family). Train two heads 这 predict start 与 end 词元 indices of the 答案. Loss is cross-entropy over 有效 positions. 输出 is a span 从 the passage. Never hallucinates (by construction), never handles questions the passage cannot 答案 (by construction).

**Retrieval-augmented (RAG).** Two stages. First, a retriever finds the top-`k` passages 从 a 语料库. Second, a reader (extractive 或 generative) produces the 答案 using those passages. The retriever-reader split lets each be trained 与 evaluated independently. 现代 RAG often adds a reranker between them.

**Generative.** A decoder-only LLM (GPT, Claude, Llama) answers 从 learned weights. No 检索 step. Excellent on 常见 knowledge, catastrophic on rare 或 recent facts. The hallucination rate is inversely correlated 使用 fact frequency in the pretraining 数据.

## 动手构建

### Step 1: extractive QA 使用 a pretrained 模型

```python
from transformers import pipeline

qa = pipeline("question-answering", model="deepset/roberta-base-squad2")

passage = (
    "Apple Inc. released the first iPhone on June 29, 2007. "
    "The device was announced by Steve Jobs at Macworld in January 2007."
)
question = "When was the first iPhone released?"

answer = qa(question=question, context=passage)
print(answer)
```

```python
{'score': 0.98, 'start': 57, 'end': 70, 'answer': 'June 29, 2007'}
```

`deepset/roberta-base-squad2` is trained on SQuAD 2.0, 这 includes unanswerable questions. By 默认, the `question-answering` 流水线 returns the highest-scoring span even when the 模型's null score wins，it does *not* automatically return an empty 答案. To get explicit "no 答案" behavior, pass `handle_impossible_answer=True` to the 流水线 call: the 流水线 then returns an empty 答案 only when the null score exceeds every span score. Always check the `score` field either way.

### Step 2: a 检索-augmented 流水线 (sketch)

```python
from sentence_transformers import SentenceTransformer
import numpy as np

encoder = SentenceTransformer("sentence-transformers/all-MiniLM-L6-v2")

corpus = [
    "Apple Inc. released the first iPhone on June 29, 2007.",
    "Macworld 2007 featured the iPhone announcement by Steve Jobs.",
    "Android launched in 2008 as Google's mobile operating system.",
    "The first iPod was released in 2001.",
]
corpus_embeddings = encoder.encode(corpus, normalize_embeddings=True)


def retrieve(question, top_k=2):
    q_emb = encoder.encode([question], normalize_embeddings=True)
    sims = (corpus_embeddings @ q_emb.T).squeeze()
    order = np.argsort(-sims)[:top_k]
    return [corpus[i] for i in order]


def answer(question):
    passages = retrieve(question, top_k=2)
    combined = " ".join(passages)
    return qa(question=question, context=combined)


print(answer("When was the first iPhone released?"))
```

Two-stage 流水线. Dense retriever (句子-BERT) finds relevant passages by 语义 相似度. Extractive reader (RoBERTa-SQuAD) pulls the 答案 span 从 the combined top passages. Works on 小 corpora. 面向 a million-文档 语料库, use FAISS 或 a 向量 database.

### Step 3: generative 使用 RAG

```python
def rag_generate(question, llm):
    passages = retrieve(question, top_k=3)
    prompt = f"""Context:
{chr(10).join('- ' + p for p in passages)}

Question: {question}

Answer using only the context above. If the context does not contain the answer, say "I don't know."
"""
    return llm(prompt)
```

说明：这个 prompt pattern matters. Explicitly telling the 模型 to ground in the context 与 return "I don't know" when the context is insufficient cuts hallucination rates by 40-60% compared to naive prompting. More elaborate patterns add citations, confidence scores, 与 structured extraction.

### Step 4: 评估 这 reflects the real world

SQuAD uses **Exact Match (EM)** 与 **词元-level F1**. EM is a strict match after normalization (lowercase, strip punctuation, remove articles)，either the prediction matches exactly 或 it scores 0. F1 is computed over 词元 overlap between prediction 与 reference 与 gives partial credit. Both under-credit paraphrases: "June 29, 2007" vs "June 29th, 2007" typically gets 0 EM (the ordinal breaks normalization) but still earns substantial F1 从 overlapping 词元.

对于 生产 QA:

- 说明：**Answer 准确率** (LLM-judged 或 human-judged, since metrics do not capture 语义 equivalence).
- **Citation 准确率.** Does the cited passage actually support the 答案? Trivial to check automatically 使用 string match between generated citations 与 retrieved passages.
- 说明：**Refusal calibration.** When the 答案 is not in the retrieved passages, does the 系统 correctly say "I don't know"? Measure false confidence rate.
- 说明：**Retrieval 召回率.** Before evaluating the reader, measure whether the retriever gets the right passage 到 the top-`k`. A reader cannot fix a missing passage.

### RAGAS: the 2026 生产 eval framework

`RAGAS` is purpose-built 面向 RAG systems 与 is the shipping 默认 in 2026. It scores four dimensions 不使用 requiring gold references:

- 说明：**Faithfulness.** Does each claim in the 答案 come 从 the retrieved context? Measured by NLI-based entailment. Your primary hallucination 指标.
- **Answer relevance.** Does the 答案 address the 问题? Measured by generating hypothetical questions 从 the 答案 与 comparing to the real 问题.
- **Context 精确率.** Of the retrieved chunks, what fraction were actually relevant? Low 精确率 = 噪声 in prompt.
- 说明：**Context 召回率.** Did the retrieved set contain all needed information? Low 召回率 = reader cannot succeed.

Reference-free scoring lets you evaluate on live 生产 traffic 不使用 curated gold answers. Layer LLM-as-judge on top 面向 开放-ended questions 其中 exact-match metrics are useless.

说明：`pip install ragas`. Plug your retriever + reader. Get four scalars per 查询. Alert on regressions.

## 投入使用

这个 2026 stack.

|Use case|推荐|
|---------|-------------|
|Given passage, find 答案 span|`deepset/roberta-base-squad2`|
|Over a 固定 语料库, 封闭-book not acceptable|RAG: dense retriever + LLM reader|
|Real-time over a 文档 store|说明：RAG 使用 hybrid (BM25 + dense) retriever + reranker (lesson 14)|
|Conversational QA (follow-up questions)|LLM 使用 conversation 历史 + RAG on each turn|
|Highly factual, regulated domains|说明：Extractive over an authoritative 语料库; never generative alone|

说明：Extractive QA is unfashionable in 2026 因为 RAG 使用 LLMs handles more cases. It still ships in contexts 其中 literal quotation is required: legal research, regulatory compliance, audit tools.

## 交付成果

保存为 `outputs/skill-qa-architect.md`:

```markdown
---
name: qa-architect
description: Choose QA architecture, retrieval strategy, and evaluation plan.
version: 1.0.0
phase: 5
lesson: 13
tags: [nlp, qa, rag]
---

Given requirements (corpus size, question type, factuality constraint, latency budget), output:

1. Architecture. Extractive, RAG with extractive reader, RAG with generative reader, or closed-book LLM. One-sentence reason.
2. Retriever. None, BM25, dense (name the encoder), or hybrid.
3. Reader. SQuAD-tuned model, LLM by name, or "domain-fine-tuned DistilBERT."
4. Evaluation. EM + F1 for extractive benchmarks; answer accuracy + citation accuracy + refusal calibration for production. Name what you are measuring and how you are measuring it.

Refuse closed-book LLM answers for regulatory or compliance-sensitive questions. Refuse any QA system without a retrieval-recall baseline (you cannot evaluate the reader without knowing the retriever surfaced the right passage). Flag questions that require multi-hop reasoning as needing specialized multi-hop retrievers like HotpotQA-trained systems.
```

## 练习

1. **Easy.** Set up the SQuAD extractive 流水线 above on 10 Wikipedia passages. Hand-craft 10 questions. Measure how often the 答案 is 正确. You should see 7-9 正确 if passages 与 questions are clean.
2. 说明：**Medium.** Add a refusal classifier. When the top 检索 score is below a threshold (say 0.3 cosine), return "I don't know" instead of calling the reader. Tune the threshold on a held-out set.
3. **Hard.** Build a RAG 流水线 over a 10,000-文档 语料库 of your choice. Implement hybrid 检索 (BM25 + dense) 使用 RRF fusion (see lesson 14). Measure 答案 准确率 使用 与 不使用 the hybrid step. 文档 这 问题 types benefit most.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|Extractive QA|Find the 答案 span|说明：Predict start 与 end indices of the 答案 within a given passage.|
|开放-领域 QA|QA over a 语料库|No given passage; must retrieve then 答案.|
|RAG|Retrieve then generate|Retrieval-augmented 生成. Retriever + reader 流水线.|
|SQuAD|Canonical 基准|说明：Stanford Question Answering Dataset. EM + F1 metrics.|
|Hallucination|Made-up 答案|说明：Reader 输出 not supported by retrieved context.|
|Refusal calibration|Know when to shut up|系统 correctly says "I don't know" when unable to 答案.|

## 延伸阅读

- [说明：Rajpurkar et al. (2016). SQuAD: 100,000+ Questions 面向 Machine Comprehension of 文本](https://arxiv.org/abs/1606.05250)，the 基准 paper.
- [说明：Karpukhin et al. (2020). Dense Passage Retrieval 面向 开放-领域 QA](https://arxiv.org/abs/2004.04906)，DPR, the canonical dense retriever 面向 QA.
- 说明：[说明：Lewis et al. (2020). Retrieval-Augmented Generation 面向 Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)，the paper 这 named RAG.
- 说明：[说明：Gao et al. (2023). Retrieval-Augmented Generation 面向 大 Language Models: A Survey](https://arxiv.org/abs/2312.10997)，comprehensive RAG survey.
