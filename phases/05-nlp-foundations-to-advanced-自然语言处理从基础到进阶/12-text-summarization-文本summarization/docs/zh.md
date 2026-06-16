# 文本摘要

> 说明：Extractive systems tell you what the 文档 said. Abstractive systems tell you what the author meant. Different tasks, different pitfalls.

**类型:** 构建
**语言:** Python
**先修要求:** 说明：Phase 5 · 02 (BoW + TF-IDF), Phase 5 · 11 (Machine Translation)
**时间:** 约75分钟

## 问题

一个 2,000-词 news article lands in your feed. You need 120 词 这 capture it. You can either pick the three most important sentences 从 the article (extractive) 或 rewrite the content in your own 词 (abstractive). Both are called summarization. They are completely different problems.

说明：Extractive summarization is a ranking problem. Score every 句子, return the top-`k`. The 输出 is always grammatical 因为 it is lifted verbatim. The risk is missing content 这 is distributed across the article.

Abstractive summarization is a 生成 problem. A transformer produces new 文本 conditioned on the input. The 输出 is fluent 与 compressive but may hallucinate facts 这 were not in the source. The risk is confident fabrication.

本课 builds both, 使用 the 失败模式 each one owns.

## 概念

说明：![说明：Extractive TextRank vs abstractive transformer](../assets/summarization.svg)

说明：**Extractive.** Treat the article as a graph 其中 nodes are sentences 与 edges are similarities. Run PageRank (或 something like it) over the graph to score sentences by how connected they are to everything else. Highest-scoring sentences are the 摘要. The canonical implementation is **TextRank** (Mihalcea 与 Tarau, 2004).

**Abstractive.** Fine-tune a transformer encoder-decoder (BART, T5, Pegasus) on 文档-摘要 pairs. At 推理, the 模型 reads the 文档 与 generates the 摘要 词元-by-词元 via cross-注意力. Pegasus in particular uses a gap-句子 pretraining objective 这 makes it excellent at summarization 不使用 much fine-tuning.

Evaluation 使用 **ROUGE** (召回率-Oriented Understudy 面向 Gisting Evaluation). ROUGE-1 与 ROUGE-2 score unigram 与 bigram overlap. ROUGE-L scores longest 常见 subsequence. Higher is better but 40 ROUGE-L is "good" 与 50 is "exceptional." Every paper reports all three. Use the `rouge-score` package.

## 动手构建

### Step 1: TextRank (extractive)

```python
import math
import re
from collections import Counter


def sentence_split(text):
    return re.split(r"(?<=[.!?])\s+", text.strip())


def similarity(s1, s2):
    w1 = Counter(s1.lower().split())
    w2 = Counter(s2.lower().split())
    intersection = sum((w1 & w2).values())
    denom = math.log(len(w1) + 1) + math.log(len(w2) + 1)
    if denom == 0:
        return 0.0
    return intersection / denom


def textrank(text, top_k=3, damping=0.85, iterations=50, epsilon=1e-4):
    sentences = sentence_split(text)
    n = len(sentences)
    if n <= top_k:
        return sentences

    sim = [[0.0] * n for _ in range(n)]
    for i in range(n):
        for j in range(n):
            if i != j:
                sim[i][j] = similarity(sentences[i], sentences[j])

    scores = [1.0] * n
    for _ in range(iterations):
        new_scores = [1 - damping] * n
        for i in range(n):
            total_out = sum(sim[i]) or 1e-9
            for j in range(n):
                if sim[i][j] > 0:
                    new_scores[j] += damping * sim[i][j] / total_out * scores[i]
        if max(abs(s - ns) for s, ns in zip(scores, new_scores)) < epsilon:
            scores = new_scores
            break
        scores = new_scores

    ranked = sorted(range(n), key=lambda k: scores[k], reverse=True)[:top_k]
    ranked.sort()
    return [sentences[i] for i in ranked]
```

Two things worth naming. The 相似度 函数 uses log-normalized 词 overlap, 这 is the original TextRank variant. Cosine of TF-IDF 向量 works too. The damping factor 0.85 与 iteration count are the PageRank defaults.

### Step 2: abstractive 使用 BART

```python
from transformers import pipeline

summarizer = pipeline("summarization", model="facebook/bart-large-cnn")

article = """(long news article text)"""

summary = summarizer(article, max_length=120, min_length=60, do_sample=False)
print(summary[0]["summary_text"])
```

BART-大-CNN is fine-tuned on the CNN/DailyMail 语料库. It produces news-style summaries out of the box. 面向 other domains (scientific papers, dialog, legal), use the corresponding Pegasus checkpoint 或 fine-tune on your target 数据.

### Step 3: ROUGE 评估

```python
from rouge_score import rouge_scorer

scorer = rouge_scorer.RougeScorer(["rouge1", "rouge2", "rougeL"], use_stemmer=True)
scores = scorer.score(reference_summary, generated_summary)
print({k: round(v.fmeasure, 3) for k, v in scores.items()})
```

Always use 词干提取. 不使用 it, "running" 与 "run" count as different 词 与 ROUGE undercounts.

### Beyond ROUGE (2026 summarization eval)

说明：ROUGE has been the dominant summarization 指标 面向 twenty years 与 it is insufficient on its own in 2026. A 大-scale meta-analysis of NLG papers showed:

- 说明：**BERTScore** (contextual 嵌入 相似度) gained ground through 2023 与 is now reported alongside ROUGE in most summarization papers.
- 说明：**BARTScore** treats 评估 as 生成: score the 摘要 by how likely a pretrained BART assigns it given the source.
- 说明：**MoverScore** (Earth Mover's Distance over contextual 嵌入) reached the top spot in 2025 summarization benchmarks 因为 it captures 语义 overlap better than ROUGE.
- 说明：**FactCC** 与 **QA-based faithfulness** were 常见 2021-2023, now often replaced by **G-Eval** (a GPT-4 prompt chain 这 scores coherence, consistency, fluency, relevance 使用 chain-of-thought reasoning).
- 说明：**G-Eval** 与 similar LLM-judge approaches match human judgment ~80% of the time when rubrics are well-designed.

Production recommendation: report ROUGE-L 面向 legacy comparison, BERTScore 面向 语义 overlap, G-Eval 面向 coherence 与 factuality. Calibrate against 50-100 human-labeled summaries.

### Step 4: the factuality problem

Abstractive summaries are prone to hallucination. Extractive summaries carry a much lower hallucination risk 因为 the 输出 is lifted verbatim 从 the source, though they can still mislead if source sentences are decontextualized, outdated, 或 quoted out of order. This is the single biggest 原因 生产 systems still prefer extractive methods 面向 compliance-adjacent content.

Hallucination types to name:

- 说明：**Entity swap.** Source says "John Smith." Summary says "John Brown."
- 说明：**Number drift.** Source says "25,000." Summary says "25 million."
- 说明：**Polarity flip.** Source says "rejected the offer." Summary says "accepted the offer."
- 说明：**Fact invention.** Source does not mention the CEO. Summary says the CEO approved.

Evaluation approaches 这 work:

- 说明：**FactCC.** A binary classifier trained on entailment between source 句子 与 摘要 句子. Predicts factual/not-factual.
- 说明：**QA-based factuality.** Ask a QA 模型 questions whose answers are in the source. If the 摘要 supports different answers, flag.
- 说明：**Entity-level F1.** Compare named entities in source vs 摘要. Entities present only in the 摘要 are suspect.

对于 anything 用户-facing 其中 factuality matters (news, medical, legal, financial), extractive is the safer 默认. Abstractive needs a factuality check in the loop.

## 投入使用

这个 2026 stack:

|Use case|推荐|
|---------|-------------|
|News, 3-5 句子 摘要, English|`facebook/bart-large-cnn`|
|Scientific papers|`google/pegasus-pubmed` 或 a tuned T5|
|Multi-文档, 长-form|Any LLM 使用 32k+ context, prompted|
|Dialog summarization|`philschmid/bart-large-cnn-samsum`|
|说明：Extractive, low hallucination risk by construction|TextRank 或 `sumy`'s LSA / LexRank|

LLMs 使用 长 context often beat specialized models in 2026 when 计算 is not a constraint. The tradeoff is 成本 与 reproducibility; specialized models give more consistent outputs.

## 交付成果

保存为 `outputs/skill-summary-picker.md`:

```markdown
---
name: summary-picker
description: Pick extractive or abstractive, named library, factuality check.
version: 1.0.0
phase: 5
lesson: 12
tags: [nlp, summarization]
---

Given a task (document type, compliance requirement, length, compute budget), output:

1. Approach. Extractive or abstractive. Explain in one sentence why.
2. Starting model / library. Name it. `sumy.TextRankSummarizer`, `facebook/bart-large-cnn`, `google/pegasus-pubmed`, or an LLM prompt.
3. Evaluation plan. ROUGE-1, ROUGE-2, ROUGE-L (use rouge-score with stemming). Plus factuality check if abstractive.
4. One failure mode to probe. Entity swap is the most common in abstractive news summarization; flag samples where source entities do not appear in summary.

Refuse abstractive summarization for medical, legal, financial, or regulated content without a factuality gate. Flag input over the model's context window as needing chunked map-reduce summarization (not just truncation).
```

## 练习

1. 说明：**Easy.** Run TextRank on 5 news articles. Compare the top-3 sentences to a reference 摘要. Measure ROUGE-L. You should see 30-45 ROUGE-L on CNN/DailyMail-style articles.
2. **Medium.** Implement 实体-level factuality: extract named entities 从 source 与 摘要 (spaCy), 计算 召回率 of source entities in 摘要 与 精确率 of 摘要 entities against source. High 精确率 与 low 召回率 mean safe but terse; low 精确率 means hallucinated entities.
3. **Hard.** Compare BART-大-CNN against an LLM (Claude 或 GPT-4) on 50 CNN/DailyMail articles. Report ROUGE-L, factuality (by 实体 F1), 与 成本 per 摘要. 文档 其中 each wins.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|Extractive|Pick sentences|说明：Return sentences verbatim 从 the source. Never hallucinates.|
|Abstractive|Rewrite|说明：Generate new 文本 conditioned on source. Can hallucinate.|
|ROUGE|Summary 指标|N-gram / LCS overlap between 系统 输出 与 reference.|
|TextRank|Graph-based extractive|PageRank over 句子 相似度 graph.|
|Factuality|Is it right|说明：Whether 摘要 claims are supported by the source.|
|Hallucination|Made-up content|Content in the 摘要 这 the source does not support.|

## 延伸阅读

- 说明：[说明：Mihalcea 与 Tarau (2004). TextRank: Bringing Order 到 Texts](https://aclanthology.org/W04-3252/)，the extractive canonical paper.
- 说明：[Lewis et al. (2019). BART: Denoising 序列-to-序列 Pre-训练](https://arxiv.org/abs/1910.13461)，the BART paper.
- [说明：Zhang et al. (2019). PEGASUS: Pre-训练 使用 Extracted Gap-sentences](https://arxiv.org/abs/1912.08777)，Pegasus 与 the gap-句子 objective.
- 说明：[说明：Lin (2004). ROUGE: A Package 面向 Automatic Evaluation of Summaries](https://aclanthology.org/W04-1013/)，ROUGE paper.
- 说明：[说明：Maynez et al. (2020). On Faithfulness 与 Factuality in Abstractive Summarization](https://arxiv.org/abs/2005.00661)，the factuality landscape paper.
