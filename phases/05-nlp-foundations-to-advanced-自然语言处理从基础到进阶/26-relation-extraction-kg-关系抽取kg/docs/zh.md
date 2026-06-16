# 关系抽取与知识图谱构建

> 说明：NER found the entities. Entity linking anchored them. Relation extraction finds the edges between them. A knowledge graph is the sum of nodes, edges, 与 their provenance.

**类型:** 构建
**语言:** Python
**先修要求:** Phase 5 · 06 (NER), Phase 5 · 25 (Entity Linking)
**时间:** 约60分钟

## 问题

说明：一个 analyst reads: "Tim Cook became CEO of Apple in 2011." Four facts:

- `(Tim Cook, role, CEO)`
- `(Tim Cook, employer, Apple)`
- `(Tim Cook, start_date, 2011)`
- `(Apple, type, Organization)`

Relation Extraction (RE) turns free 文本 到 structured triples `(subject, relation, object)`. Aggregate across a 语料库 与 you have a knowledge graph. Aggregate 与 查询 与 you have a reasoning substrate 面向 RAG, analytics, 或 compliance audits.

这个 2026 problem: LLMs extract relations enthusiastically. Too enthusiastically. They hallucinate triples 这 the source 文本 does not support. 不使用 provenance, you cannot tell real triples 从 plausible fiction. The 2026 答案 is AEVS-style anchor-与-verify pipelines.

## 概念

说明：![文本 → triples → knowledge graph](../assets/relation-extraction.svg)

说明：**Triple form.** `(subject_entity, relation_type, object_entity)`. Relations come 从 a 封闭 ontology (Wikidata properties, FIBO, UMLS) 或 an 开放 set (OpenIE-style, anything goes).

**Three extraction approaches.**

1. 说明：**Rule / pattern-based.** Hearst patterns: "X such as Y" → `(Y, isA, X)`. Plus hand-crafted regex. Brittle, precise, explainable.
2. **Supervised classifier.** Given two 实体 mentions in a 句子, predict the 关系 从 a 固定 set. Trained on TACRED, ACE, KBP. Standard 2015–2022.
3. 说明：**Generative LLM.** Prompt the 模型 to emit triples. Works out of the box. Needs provenance, 或 hallucinates plausible-looking junk.

说明：**AEVS (Anchor-Extraction-Verification-Supplement, 2026).** The 当前 hallucination-mitigation framework:

- 说明：**Anchor.** Identify every 实体 span 与 关系-phrase span 使用 exact positions.
- 说明：**Extract.** Generate triples linked to anchor spans.
- 说明：**Verify.** Match each triple element back to the source 文本; reject anything unsupported.
- 说明：**Supplement.** A coverage pass ensures no anchored span is dropped.

说明：Hallucinations drop sharply. Requires more 计算 but is auditable.

**The 开放-vs-封闭 tradeoff.**

- 说明：**封闭 ontology.** 固定 property list (e.g., Wikidata's 11,000+ properties). Predictable. Queryable. Hard to invent.
- **开放 IE.** Any verbal phrase becomes a 关系. High 召回率. Low 精确率. Messy to 查询.

说明：Production KGs usually mix: 开放 IE 面向 discovery, then canonicalize relations onto a 封闭 ontology before merging 到 the main graph.

## 动手构建

### Step 1: pattern-based extraction

```python
PATTERNS = [
    (r"(?P<s>[A-Z]\w+) (?:is|was) (?:a|an|the) (?P<o>[A-Z]?\w+)", "isA"),
    (r"(?P<s>[A-Z]\w+) (?:is|was) born in (?P<o>\w+)", "bornIn"),
    (r"(?P<s>[A-Z]\w+) works? (?:at|for) (?P<o>[A-Z]\w+)", "worksAt"),
    (r"(?P<s>[A-Z]\w+) founded (?P<o>[A-Z]\w+)", "founded"),
]
```

说明：See `code/main.py` 面向 the full toy extractor. Hearst patterns still ship in 领域-specific pipelines 因为 they are debuggable.

### Step 2: supervised 关系 分类

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification

tok = AutoTokenizer.from_pretrained("Babelscape/rebel-large")
model = AutoModelForSequenceClassification.from_pretrained("Babelscape/rebel-large")

text = "Tim Cook was born in Alabama. He later became CEO of Apple."
encoded = tok(text, return_tensors="pt", truncation=True)
output = model.generate(**encoded, max_length=200)
triples = tok.batch_decode(output, skip_special_tokens=False)
```

REBEL is a seq2seq 关系 extractor: 文本 in, triples out, already in Wikidata property ids. Fine-tuned on distant-supervision 数据. Standard 开放-weights 基线.

### Step 3: LLM-prompted extraction 使用 anchoring

```python
prompt = f"""Extract (subject, relation, object) triples from the text.
For each triple, include the exact character span in the source text.

Text: {text}

Output JSON:
[{{"subject": {{"text": "...", "span": [start, end]}},
   "relation": "...",
   "object": {{"text": "...", "span": [start, end]}}}}, ...]

Only include triples fully supported by the text. No inference beyond what is stated.
"""
```

说明：Verify every returned span against the source. Reject anything 其中 `text[start:end] != triple_entity`. This is the AEVS "verify" step in its minimal form.

### Step 4: canonicalize onto a 封闭 ontology

```python
RELATION_MAP = {
    "is the CEO of": "P169",       # "chief executive officer"
    "was born in":   "P19",         # "place of birth"
    "founded":        "P112",       # "founded by" (inverted subject/object)
    "works at":       "P108",       # "employer"
}


def canonicalize(relation):
    rel_low = relation.lower().strip()
    if rel_low in RELATION_MAP:
        return RELATION_MAP[rel_low]
    return None   # drop unmapped open relations or route to manual review
```

说明：Canonicalization is often 60-80% of the engineering work. 预算 面向 it.

### Step 5: build a 小 graph 与 查询

```python
triples = extract(text)
graph = {}
for s, r, o in triples:
    graph.setdefault(s, []).append((r, o))


def neighbors(node, relation=None):
    return [(r, o) for r, o in graph.get(node, []) if relation is None or r == relation]


print(neighbors("Tim Cook", relation="P108"))    # -> [(P108, Apple)]
```

说明：This is the atom of every RAG-over-KG 系统. Scale it 使用 RDF triple stores (Blazegraph, Virtuoso), property graphs (Neo4j), 或 向量-augmented graph stores.

## 陷阱

- 说明：**Coreference before RE.** "He founded Apple"，RE needs to know who "he" is. Run coref first (lesson 24).
- 说明：**Entity canonicalization.** "Apple Inc" 与 "Apple" must resolve to the same node. Entity linking first (lesson 25).
- 说明：**Hallucinated triples.** LLMs emit triples the 文本 does not support. Enforce span verification.
- 说明：**Relation canonicalization drift.** 开放 IE relations are inconsistent ("was born in," "came 从," "is a native of"). Collapse to canonical ids 或 the graph is unqueryable.
- 说明：**Temporal errors.** "Tim Cook is CEO of Apple"，true now, false in 2005. Many relations are time-bounded. Use qualifiers (`P580` start time, `P582` end time in Wikidata).
- 说明：**领域 mismatch.** REBEL trained on Wikipedia. Legal, medical, 与 scientific 文本 often need 领域-fine-tuned RE models.

## 投入使用

这个 2026 stack:

|Situation|Pick|
|-----------|------|
|快 生产, general 领域|说明：REBEL 或 LlamaPred 使用 Wikidata canonicalization|
|领域-specific (biomed, legal)|SciREX-style 领域 fine-tune + custom ontology|
|LLM-prompted, audited 输出|AEVS 流水线: anchor → extract → verify → supplement|
|High-volume news IE|Pattern-based + supervised hybrid|
|Building a KG 从 scratch|开放 IE + manual canonicalization pass|
|Temporal KG|说明：Extract 使用 qualifiers (start/end time, point in time)|

这个 integration pattern: NER → coref → 实体 linking → 关系 extraction → ontology mapping → graph load. Every stage is a potential 质量 gate.

## 交付成果

保存为 `outputs/skill-re-designer.md`:

```markdown
---
name: re-designer
description: Design a relation extraction pipeline with provenance and canonicalization.
version: 1.0.0
phase: 5
lesson: 26
tags: [nlp, relation-extraction, knowledge-graph]
---

Given a corpus (domain, language, volume) and downstream use (KG-RAG, analytics, compliance), output:

1. Extractor. Pattern-based / supervised / LLM / AEVS hybrid. Reason tied to precision vs recall target.
2. Ontology. Closed property list (Wikidata / domain) or open IE with canonicalization pass.
3. Provenance. Every triple carries source char-span + doc id. Non-negotiable for audit.
4. Merge strategy. Canonical entity id + relation id + temporal qualifiers; dedup policy.
5. Evaluation. Precision / recall on 200 hand-labelled triples + hallucination-rate on LLM-extracted sample.

Refuse any LLM-based RE pipeline without span verification (source provenance). Refuse open-IE output flowing into a production graph without canonicalization. Flag pipelines with no temporal qualifier on time-bounded relations (employer, spouse, position).
```

## 练习

1. 说明：**Easy.** Run the pattern extractor in `code/main.py` on 5 news-article sentences. Hand-check 精确率.
2. **Medium.** Use REBEL (或 a 小 LLM) on the same sentences. Compare triples. 这 extractor has higher 精确率? Higher 召回率?
3. 说明：**Hard.** Build the AEVS 流水线: extract 使用 LLM + verify spans against source. Measure hallucination rate before vs after the verify step on 50 Wikipedia-style sentences.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|Triple|Subject-关系-object|`(s, r, o)` tuple 这 is the atomic unit of a KG.|
|开放 IE|Extract anything|开放-vocabulary 关系 phrases; high 召回率, low 精确率.|
|封闭 ontology|固定 模式|Bounded set of 关系 types (Wikidata, UMLS, FIBO).|
|Canonicalization|Normalize everything|说明：Map surface names / relations to canonical ids.|
|AEVS|Grounded extraction|说明：Anchor-Extraction-Verification-Supplement 流水线 (2026).|
|Provenance|Source-of-truth link|说明：每个 triple carries a doc id + char-span to its source.|
|Distant supervision|Cheap labels|Align 文本 使用 an existing KG to create 训练 数据.|

## 延伸阅读

- [Mintz et al. (2009). Distant supervision 面向 关系 extraction 不使用 labeled 数据](https://www.aclweb.org/anthology/P09-1113.pdf)，the distant-supervision paper.
- 说明：[说明：Huguet Cabot, Navigli (2021). REBEL: Relation Extraction By End-to-end Language 生成](https://aclanthology.org/2021.findings-emnlp.204.pdf)，seq2seq RE workhorse.
- 说明：[说明：Wadden et al. (2019). Entity, Relation, 与 Event Extraction 使用 Contextualized Span Representations (DyGIE++)](https://arxiv.org/abs/1909.03546)，joint IE.
- 说明：[说明：AEVS，Anchor-Extraction-Verification-Supplement framework](https://www.mdpi.com/2073-431X/15/3/178)，2026 hallucination-mitigation design.
- 说明：[Wikidata SPARQL tutorial](https://www.wikidata.org/wiki/Wikidata:SPARQL_tutorial)，canonical graph queries.
