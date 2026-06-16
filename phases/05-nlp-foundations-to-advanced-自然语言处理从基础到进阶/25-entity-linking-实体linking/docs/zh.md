# 实体链接与消歧

> 说明：NER found "Paris." Entity linking decides: Paris, France? Paris Hilton? Paris, Texas? Paris (the Trojan prince)? 不使用 linking, your knowledge graph stays ambiguous.

**类型:** 构建
**语言:** Python
**先修要求:** Phase 5 · 06 (NER), Phase 5 · 24 (Coreference Resolution)
**时间:** 约60分钟

## 问题

说明：一个 句子 reads: "Jordan beat the press." Your NER tags "Jordan" as PERSON. Good. But *这* Jordan?

- Michael Jordan (basketball)?
- Michael B. Jordan (actor)?
- 说明：Michael I. Jordan (Berkeley ML professor，yes, this confusion is real in ML papers)?
- Jordan (the country)?
- Jordan (Hebrew first name)?

说明：Entity linking (EL) resolves each mention to a unique entry in a knowledge base: Wikidata, Wikipedia, DBpedia, 或 your 领域 KB. Two subtasks:

1. 说明：**Candidate 生成.** Given "Jordan," 这 KB entries are plausible?
2. 说明：**Disambiguation.** 给定 context, 这 candidate is the right one?

两者都 steps are learnable. Both are benchmarked. The combined 流水线 has been stable 面向 a decade，what changes is the 质量 of the disambiguator.

## 概念

![说明：Entity linking 流水线: mention → candidates → disambiguated 实体](../assets/entity-linking.svg)

说明：**Candidate 生成.** 给定 mention surface form ("Jordan"), look up candidates in an alias index. Wikipedia alias dictionaries cover most named entities: "JFK" → John F. Kennedy, Jacqueline Kennedy, JFK airport, JFK (movie). Typical index returns 10-30 candidates per mention.

**Disambiguation: three approaches.**

1. **Prior + context (Milne & Witten, 2008).** `P(entity | mention) × context-similarity(entity, text)`. Works well, 快, no 训练.
2. 说明：**嵌入-based (ESS / REL / Blink).** Encode mention + context. Encode each candidate's description. Pick max cosine. The 2020-2024 默认.
3. **Generative (GENRE, 2021; LLM-based, 2023+).** Decode the 实体's canonical name 词元-by-词元. Constrained to a trie of 有效 实体 names so 输出 is guaranteed to be a 有效 KB id.

**End-to-end vs 流水线.** 现代 models (ELQ, BLINK, ExtEnD, GENRE) run NER + candidate 生成 + disambiguation in one pass. Pipeline systems still dominate in 生产 因为 you can swap components.

### 这个 two measurements

- **Mention 召回率 (candidate gen).** Fraction of gold mentions 其中 the 正确 KB entry appears in the candidate list. Floor 面向 the whole 流水线.
- 说明：**Disambiguation 准确率 / F1.** Given 正确 candidates, how often the top-1 is right.

Always report both. A 系统 使用 99% disambiguation on 80% candidate 召回率 is an 80% 流水线.

## 动手构建

### 说明：Step 1: build an alias index 从 Wikipedia redirects

```python
alias_to_entities = {
    "jordan": ["Q41421 (Michael Jordan)", "Q810 (Jordan, country)", "Q254110 (Michael B. Jordan)"],
    "paris":  ["Q90 (Paris, France)", "Q663094 (Paris, Texas)", "Q55411 (Paris Hilton)"],
    "apple":  ["Q312 (Apple Inc.)", "Q89 (apple, fruit)"],
}
```

说明：Wikipedia alias 数据: ~18M (alias, 实体) pairs. Download 从 Wikidata dumps. Store as inverted index.

### Step 2: context-based disambiguation

```python
def disambiguate(mention, context, alias_index, entity_desc):
    candidates = alias_index.get(mention.lower(), [])
    if not candidates:
        return None, 0.0
    context_words = set(tokenize(context))
    best, best_score = None, -1
    for entity_id in candidates:
        desc_words = set(tokenize(entity_desc[entity_id]))
        union = len(context_words | desc_words)
        score = len(context_words & desc_words) / union if union else 0.0
        if score > best_score:
            best, best_score = entity_id, score
    return best, best_score
```

这个 Jaccard overlap is a toy. Replace 使用 cosine 相似度 on 嵌入 (see `code/main.py` step-2 面向 the transformer version).

### Step 3: 嵌入-based (BLINK-style)

```python
from sentence_transformers import SentenceTransformer
encoder = SentenceTransformer("sentence-transformers/all-MiniLM-L6-v2")

def embed_mention(text, mention_span):
    start, end = mention_span
    marked = f"{text[:start]} [MENTION] {text[start:end]} [/MENTION] {text[end:]}"
    return encoder.encode([marked], normalize_embeddings=True)[0]

def embed_entity(entity_id, description):
    return encoder.encode([f"{entity_id}: {description}"], normalize_embeddings=True)[0]
```

说明：At index time, embed every KB 实体 once. At 查询 time, embed the mention + context once, dot-product against the candidate pool, pick max.

### Step 4: generative 实体 linking (concept)

GENRE decodes the 实体's Wikipedia title character-by-character. Constrained decoding (see lesson 20) ensures only 有效 titles can be 输出. Tight integration 使用 a KB-backed trie. The 现代 descendant is REL-GEN 与 LLM-prompted EL 使用 structured 输出.

```python
prompt = f"""Text: {text}
Mention: {mention}
List the best Wikipedia title for this mention.
Respond with JSON: {{"title": "..."}}"""
```

说明：Combined 使用 a whitelist (Outlines `choice`), this is the simplest EL 流水线 to ship in 2026.

### Step 5: evaluate on AIDA-CoNLL

说明：AIDA-CoNLL is the standard EL 基准: 1,393 Reuters articles, 34k mentions, Wikipedia entities. Report in-KB 准确率 (`P@1`) 与 out-of-KB NIL-detection rate.

## 陷阱

- 说明：**NIL handling.** Some mentions are not in the KB (emerging entities, obscure people). Systems must predict NIL instead of guessing the 错误 实体. Measured separately.
- 说明：**Mention boundary errors.** Upstream NER misses partial spans ("Bank of America" tagged as just "Bank"). EL 召回率 drops.
- 说明：**Popularity bias.** Trained systems over-predict frequent entities. A mention of "Michael I. Jordan" on an ML paper often links to basketball Jordan.
- **Cross-lingual EL.** Mapping mentions in Chinese 文本 to English Wikipedia entities. Requires a 多语言 encoder 或 a 翻译 step.
- 说明：**KB staleness.** New companies, events, people are not in last year's Wikipedia dump. Production pipelines need a refresh loop.

## 投入使用

这个 2026 stack:

|Situation|Pick|
|-----------|------|
|General-purpose English + Wikipedia|BLINK 或 REL|
|Cross-lingual, KB = Wikipedia|mGENRE|
|LLM-friendly, few mentions/day|说明：Prompt Claude/GPT-4 使用 candidate list + constrained JSON|
|领域-specific KB (medical, legal)|说明：Custom BERT 使用 KB-aware 检索 + fine-tune on 领域 AIDA-style set|
|Extremely low-延迟|Exact-match prior only (Milne-Witten 基线)|
|Research SOTA|GENRE / ExtEnD / generative LLM-EL|

Production pattern 这 ships in 2026: NER → coref → EL on each mention → collapse clusters to one canonical 实体 per 聚类. 输出: one KB id per 实体 in the 文档, not one per mention.

## 交付成果

保存为 `outputs/skill-entity-linker.md`:

```markdown
---
name: entity-linker
description: Design an entity linking pipeline — KB, candidate generator, disambiguator, evaluation.
version: 1.0.0
phase: 5
lesson: 25
tags: [nlp, entity-linking, knowledge-graph]
---

Given a use case (domain KB, language, volume, latency budget), output:

1. Knowledge base. Wikidata / Wikipedia / custom KB. Version date. Refresh cadence.
2. Candidate generator. Alias-index, embedding, or hybrid. Target mention recall @ K.
3. Disambiguator. Prior + context, embedding-based, generative, or LLM-prompted.
4. NIL strategy. Threshold on top score, classifier, or explicit NIL candidate.
5. Evaluation. Mention recall @ 30, top-1 accuracy, NIL-detection F1 on held-out set.

Refuse any EL pipeline without a mention-recall baseline (you cannot evaluate a disambiguator without knowing candidate gen surfaced the right entity). Refuse any pipeline using LLM-prompted EL without constrained output to valid KB ids. Flag systems where popularity bias affects minority entities (e.g. name-clashes) without domain fine-tuning.
```

## 练习

1. **Easy.** Implement the prior+context disambiguator in `code/main.py` on 10 ambiguous mentions (Paris, Jordan, Apple). Hand-标签 the 正确 实体. Measure 准确率.
2. 说明：**Medium.** Encode 50 ambiguous mentions 使用 a 句子 transformer. Embed each candidate's description. Compare 嵌入-based disambiguation to Jaccard context overlap.
3. **Hard.** Build a 1k-实体 领域 KB (e.g. employees + products in your company). Implement NER + EL end-to-end. Measure 精确率 与 召回率 on 100 held-out sentences.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|Entity linking (EL)|Link to Wikipedia|Map a mention to a unique KB entry.|
|Candidate 生成|Who could it be?|说明：Return a shortlist of plausible KB entries 面向 a mention.|
|Disambiguation|Pick the right one|说明：Score candidates using context, pick the winner.|
|Alias index|这个 lookup table|Map 从 surface form → candidate entities.|
|NIL|Not in KB|Explicit prediction 这 no KB entry matches.|
|KB|Knowledge base|Wikidata, Wikipedia, DBpedia, 或 your 领域 KB.|
|AIDA-CoNLL|这个 基准|1,393 Reuters articles 使用 gold 实体 links.|

## 延伸阅读

- 说明：[Milne, Witten (2008). Learning to Link 使用 Wikipedia](https://www.cs.waikato.ac.nz/~ihw/papers/08-DM-IHW-LearningToLinkWithWikipedia.pdf)，the foundational prior+context approach.
- 说明：[说明：Wu et al. (2020). Zero-shot Entity Linking 使用 Dense Entity Retrieval (BLINK)](https://arxiv.org/abs/1911.03814)，the 嵌入-based workhorse.
- 说明：[说明：De Cao et al. (2021). Autoregressive Entity Retrieval (GENRE)](https://arxiv.org/abs/2010.00904)，generative EL 使用 constrained decoding.
- 说明：[说明：Hoffart et al. (2011). Robust Disambiguation of Named Entities in 文本 (AIDA)](https://www.aclweb.org/anthology/D11-1072.pdf)，the 基准 paper.
- 说明：[说明：REL: An Entity Linker Standing on the Shoulders of Giants (2020)](https://arxiv.org/abs/2006.01969)，the 开放 生产 stack.
