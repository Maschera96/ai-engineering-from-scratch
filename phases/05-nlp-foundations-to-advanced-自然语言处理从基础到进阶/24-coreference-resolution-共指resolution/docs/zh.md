# 共指消解

> 说明："She called him. He did not 答案. The doctor was at lunch." Three references to two people 与 nobody is named. Coreference resolution figures out who is who.

**类型:** 学习
**语言:** Python
**先修要求:** Phase 5 · 06 (NER), Phase 5 · 07 (POS & Parsing)
**时间:** 约60分钟

## 问题

Extract every mention of Apple Inc. 从 a 300-词 article. Easy when the article says "Apple." Hard when it says "the company," "they," "Cupertino's technology giant," 或 "Jobs's firm." 不使用 resolving these mentions to the same 实体, your NER 流水线 misses 60-80% of the mentions.

说明：Coreference resolution links every expression 这 refers to the same real-world 实体 到 one 聚类. It is the glue between surface-level NLP (NER, parsing) 与 downstream semantics (IE, QA, summarization, KG).

为什么 it matters in 2026:

- 说明：Summarization: "The CEO announced..." vs "Tim Cook announced..."，the 摘要 should name the CEO.
- 说明：Question answering: "Who did she call?" requires resolving "she."
- 说明：Information extraction: a knowledge graph 使用 "PER1 founded Apple" 与 "Jobs founded Apple" as separate entries is 错误.
- 说明：Multi-文档 IE: merging mentions across articles about the same event is cross-文档 coreference.

## 概念

说明：![说明：Coreference clustering: mentions → entities](../assets/coref.svg)

**The 任务.** Input: a 文档. 输出: a clustering of mentions (spans) 其中 each 聚类 refers to one 实体.

**Mention types.**

- **Named 实体.** "Tim Cook"
- **Nominal.** "the CEO", "the company"
- **Pronominal.** "he", "she", "they", "it"
- **Appositive.** "Tim Cook, Apple's CEO,"

**Architectures.**

1. 说明：**Rule-based (Hobbs, 1978).** Syntactic-tree-based pronoun resolution using grammar rules. Good 基线. Surprisingly hard to beat on pronouns.
2. 说明：**Mention-pair classifier.** 面向 every pair of mentions (m_i, m_j), predict whether they corefer. 聚类 by transitive closure. Standard pre-2016.
3. 说明：**Mention-ranking.** 面向 each mention, rank candidate antecedents (including "no antecedent"). Pick the top.
4. **Span-based end-to-end (Lee et al., 2017).** Transformer encoder. Enumerate all candidate spans up to a length cap. Predict mention scores. Predict antecedent-概率 面向 each span. 聚类 greedily. The 现代 默认.
5. 说明：**Generative (2024+).** Prompt an LLM: "List every pronoun in this 文本 与 its antecedent." Works well on easy cases, struggles on 长 文档 与 rare referents.

**The 评估 metrics.** Five standard metrics (MUC, B³, CEAF, BLANC, LEA) 因为 no single 指标 captures clustering 质量. Report the average of the first three as CoNLL F1. State-of-the-art in 2026 on CoNLL-2012: ~83 F1.

**Known hard cases.**

- 说明：Definite descriptions referring to entities introduced pages earlier.
- 说明：Bridging anaphora ("the wheels" → a previously mentioned car).
- 说明：Zero anaphora in languages like Chinese 与 Japanese.
- 说明：Cataphora (pronoun before referent): "When **she** walked in, Mary smiled."

## 动手构建

### 说明：Step 1: pretrained neural coreference (AllenNLP / spaCy-experimental)

```python
import spacy
nlp = spacy.load("en_coreference_web_trf")   # experimental model
doc = nlp("Apple announced new products. The company said they would ship soon.")
for cluster in doc._.coref_clusters:
    print(cluster, "->", [m.text for m in cluster])
```

On a longer 文档, you get something like:
- 聚类 1: [Apple, The company, they]
- 聚类 2: [new products]

### 说明：Step 2: rule-based pronoun resolver (teaching)

See `code/main.py` 面向 a stdlib-only implementation:

1. 说明：Extract mentions: named entities (capitalized spans), pronouns (dict lookup), definite descriptions ("the X").
2. 说明：对于 each pronoun, look at the previous K mentions 与 score them by:
   - gender/number agreement (heuristic)
   - recency (closer wins)
   - syntactic role (subjects preferred)
3. Link the highest-scoring antecedent.

说明：Not competitive 使用 neural models. But it shows the search space 与 the decisions an end-to-end 模型 must make.

### Step 3: using LLMs 面向 coreference

```python
prompt = f"""Text: {text}

List every pronoun and noun phrase that refers to a person or company.
Cluster them by what they refer to. Output JSON:
[{{"entity": "Apple", "mentions": ["Apple", "the company", "it"]}}, ...]
"""
```

说明：Two failure modes to watch. First, LLMs over-merge ("him" 与 "her" referring to two distinct people). Second, LLMs silently drop mentions in 长 文档. Always verify 使用 span-offset checks.

### Step 4: 评估

这个 standard conll-2012 script computes MUC, B³, CEAF-φ4 与 reports the average. 面向 an in-house eval, start 使用 span-level 精确率 与 召回率 on your annotated test set, then add mention-linking F1.

## 陷阱

- 说明：**Singleton explosion.** Some systems report every mention as its own 聚类. B³ is lenient. MUC punishes this. Always check all three metrics.
- 说明：**Pronouns in 长 context.** Performance drops ~15 F1 on 文档 over 2,000 词元. Chunk carefully.
- 说明：**Gender assumptions.** Hard-coded gender rules break on non-binary referents, organizations, animals. Use learned models 或 neutral scoring.
- 说明：**LLM drift on 长 docs.** A single API call cannot reliably 聚类 mentions across 50+ paragraphs. Use sliding-window + merge.

## 投入使用

这个 2026 stack:

|Situation|Pick|
|-----------|------|
|English, single 文档|说明：`en_coreference_web_trf` (spaCy-experimental) 或 AllenNLP neural coref|
|Multilingual|说明：SpanBERT / XLM-R trained on OntoNotes 或 Multilingual CoNLL|
|Cross-文档 event coref|Specialized end-to-end models (2025–26 SOTA)|
|Quick LLM 基线|GPT-4o / Claude 使用 structured-输出 coref prompt|
|Production dialog systems|说明：Rule-based fallback + neural primary + manual review 面向 critical 槽位|

这个 integration pattern 这 ships in 2026: run NER first, run coref, merge coref clusters 到 NER entities. Downstream tasks see one 实体 per 聚类, not one 实体 per mention.

## 交付成果

保存为 `outputs/skill-coref-picker.md`:

```markdown
---
name: coref-picker
description: Pick a coreference approach, evaluation plan, and integration strategy.
version: 1.0.0
phase: 5
lesson: 24
tags: [nlp, coref, information-extraction]
---

Given a use case (single-doc / multi-doc, domain, language), output:

1. Approach. Rule-based / neural span-based / LLM-prompted / hybrid. One-sentence reason.
2. Model. Named checkpoint if neural.
3. Integration. Order of operations: tokenize → NER → coref → downstream task.
4. Evaluation. CoNLL F1 (MUC + B³ + CEAF-φ4 average) on held-out set + manual cluster review on 20 documents.

Refuse LLM-only coref for documents over 2,000 tokens without sliding-window merge. Refuse any pipeline that runs coref without a mention-level precision-recall report. Flag gender-heuristic systems deployed in demographically diverse text.
```

## 练习

1. 说明：**Easy.** Run the rule-based resolver in `code/main.py` on 5 hand-crafted paragraphs. Measure mention-link 准确率 against ground truth.
2. 说明：**Medium.** Use a pretrained neural coref 模型 on a news article. Compare clusters against your own manual annotation. 其中 did it fail?
3. 说明：**Hard.** Build a coref-enhanced NER 流水线: NER first, then merge via coref clusters. Measure 实体-coverage improvement vs NER-only on 100 articles.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|Mention|一个 reference|说明：一个 span of 文本 这 refers to an 实体 (name, pronoun, noun phrase).|
|Antecedent|What "it" refers to|这个 earlier mention a later one corefers 使用.|
|聚类|这个 实体's mentions|说明：Set of mentions 这 all refer to the same real-world 实体.|
|Anaphora|Backward reference|Later mention refers to earlier ("he" → "John").|
|Cataphora|Forward reference|说明：Earlier mention refers to later ("When he arrived, John...").|
|Bridging|Implicit reference|说明："I bought a car. The wheels were bad." (wheels of 这 car.)|
|CoNLL F1|这个 number on leaderboards|Average of MUC, B³, CEAF-φ4 F1 scores.|

## 延伸阅读

- 说明：[说明：Jurafsky & Martin, SLP3 Ch. 26，Coreference Resolution 与 Entity Linking](https://web.stanford.edu/~jurafsky/slp3/26.pdf)，canonical textbook chapter.
- 说明：[说明：Lee et al. (2017). End-to-end Neural Coreference Resolution](https://arxiv.org/abs/1707.07045)，span-based end-to-end.
- 说明：[Joshi et al. (2020). SpanBERT](https://arxiv.org/abs/1907.10529)，pretraining 这 improves coref.
- [Pradhan et al. (2012). CoNLL-2012 Shared 任务](https://aclanthology.org/W12-4501/)，the 基准.
- 说明：[Hobbs (1978). Resolving Pronoun References](https://www.sciencedirect.com/science/article/pii/0024384178900064)，the rule-based 经典.
