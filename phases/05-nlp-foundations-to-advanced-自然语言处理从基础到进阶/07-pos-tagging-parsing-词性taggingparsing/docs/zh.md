# 词性标注与句法解析

> 说明：Grammar was unfashionable 面向 a while. Then every LLM 流水线 needed to validate structured extraction, 与 it came back.

**类型:** 构建
**语言:** Python
**先修要求:** Phase 5 · 01 (文本 Processing), Phase 2 · 14 (Naive Bayes)
**时间:** 约45分钟

## 问题

Lesson 01 promised 这 词形还原 needs a part-of-speech tag. 不使用 knowing `running` is a verb, a lemmatizer cannot reduce it to `run`. 不使用 knowing `better` is an adjective, it cannot reduce to `good`.

这 promise hid a whole subfield. Part-of-speech tagging assigns grammatical categories. Syntactic parsing recovers the 句子's tree structure: 这 词 modifies 这, 这 verb governs 这 arguments. Classical NLP spent twenty years refining both. Then deep learning collapsed them 到 a 词元-分类 任务 on top of a pretrained transformer, 与 the research community moved on.

Not the applied community. Every structured-extraction 流水线 still uses POS 与 dependency trees under the hood. LLM-generated JSON gets validated against grammatical constraints. Question-answering systems decompose queries using dependency parses. Machine 翻译 质量 evaluators check alignment of parse trees.

说明：Worth knowing. This lesson introduces the tagsets, the baselines, 与 the point 其中 you stop implementing 从 scratch 与 call spaCy.

## 概念

**POS tagging** labels each 词元 使用 a grammatical category. The **Penn Treebank (PTB)** tagset is the English 默认. 36 tags 使用 distinctions the casual reader finds fussy: `NN` singular noun, `NNS` plural noun, `NNP` proper noun singular, `VBD` verb past tense, `VBZ` verb 3rd person singular present, 与 so on. The **Universal Dependencies (UD)** tagset is coarser (17 tags) 与 language-agnostic; it became the 默认 面向 cross-lingual work.

```
The/DET cats/NOUN were/AUX running/VERB at/ADP 3pm/NOUN ./PUNCT
```

说明：**Syntactic parsing** produces a tree. Two major styles:

- 说明：**Constituency parsing.** Noun phrases, verb phrases, prepositional phrases nest inside each other. 输出 is a tree of non-terminal categories (NP, VP, PP) 使用 词 as leaves.
- **Dependency parsing.** Each 词 has a single head 词 it depends on, labeled 使用 a grammatical 关系. 输出 is a tree 其中 every 边缘 is a (head, dependent, 关系) triple.

说明：Dependency parsing won in the 2010s 因为 it generalizes cleanly across languages, especially free-词-order ones.

```
running is ROOT
cats is nsubj of running
were is aux of running
at is prep of running
3pm is pobj of at
```

## 动手构建

### Step 1: most-frequent-tag 基线

这个 dumbest POS tagger 这 works. 面向 each 词, predict the tag it had most often in 训练.

```python
from collections import Counter, defaultdict


def train_mft(train_examples):
    word_tag_counts = defaultdict(Counter)
    all_tags = Counter()
    for tokens, tags in train_examples:
        for token, tag in zip(tokens, tags):
            word_tag_counts[token.lower()][tag] += 1
            all_tags[tag] += 1
    word_best = {w: c.most_common(1)[0][0] for w, c in word_tag_counts.items()}
    default_tag = all_tags.most_common(1)[0][0]
    return word_best, default_tag


def predict_mft(tokens, word_best, default_tag):
    return [word_best.get(t.lower(), default_tag) for t in tokens]
```

On the Brown 语料库, this 基线 hits ~85% 准确率. Not good, but the floor below 这 no serious 模型 should fall.

### Step 2: bigram HMM tagger

模型 the joint 概率 of the 序列:

```
P(tags, words) = prod P(tag_i | tag_{i-1}) * P(word_i | tag_i)
```

说明：Two tables: transition probabilities (tag given previous tag), emission probabilities (词 given tag). Estimate both 从 counts 使用 Laplace smoothing. Decode 使用 Viterbi (dynamic programming over the tag lattice).

```python
import math


def train_hmm(train_examples, alpha=0.01):
    transitions = defaultdict(Counter)
    emissions = defaultdict(Counter)
    tags = set()
    vocab = set()

    for tokens, ts in train_examples:
        prev = "<BOS>"
        for token, tag in zip(tokens, ts):
            transitions[prev][tag] += 1
            emissions[tag][token.lower()] += 1
            tags.add(tag)
            vocab.add(token.lower())
            prev = tag
        transitions[prev]["<EOS>"] += 1

    return transitions, emissions, tags, vocab


def log_prob(table, given, key, smooth_denom, alpha):
    return math.log((table[given].get(key, 0) + alpha) / smooth_denom)


def viterbi(tokens, transitions, emissions, tags, vocab, alpha=0.01):
    tags_list = list(tags)
    n = len(tokens)
    V = [[0.0] * len(tags_list) for _ in range(n)]
    back = [[0] * len(tags_list) for _ in range(n)]

    for j, tag in enumerate(tags_list):
        em_denom = sum(emissions[tag].values()) + alpha * (len(vocab) + 1)
        tr_denom = sum(transitions["<BOS>"].values()) + alpha * (len(tags_list) + 1)
        tr = log_prob(transitions, "<BOS>", tag, tr_denom, alpha)
        em = log_prob(emissions, tag, tokens[0].lower(), em_denom, alpha)
        V[0][j] = tr + em
        back[0][j] = 0

    for i in range(1, n):
        for j, tag in enumerate(tags_list):
            em_denom = sum(emissions[tag].values()) + alpha * (len(vocab) + 1)
            em = log_prob(emissions, tag, tokens[i].lower(), em_denom, alpha)
            best_prev = 0
            best_score = -1e30
            for k, prev_tag in enumerate(tags_list):
                tr_denom = sum(transitions[prev_tag].values()) + alpha * (len(tags_list) + 1)
                tr = log_prob(transitions, prev_tag, tag, tr_denom, alpha)
                score = V[i - 1][k] + tr + em
                if score > best_score:
                    best_score = score
                    best_prev = k
            V[i][j] = best_score
            back[i][j] = best_prev

    last_best = max(range(len(tags_list)), key=lambda j: V[n - 1][j])
    path = [last_best]
    for i in range(n - 1, 0, -1):
        path.append(back[i][path[-1]])
    return [tags_list[j] for j in reversed(path)]
```

Bigram HMM on Brown hits ~93% 准确率. The jump 从 85% to 93% is mostly transition probabilities，the 模型 learns `DET NOUN` is 常见 与 `NOUN DET` is rare.

### Step 3: why 现代 taggers beat this

Transition + emission probabilities are local. They cannot capture 这 `saw` is a noun in "I bought a saw" but a verb in "I saw the movie." A CRF 使用 arbitrary features (suffix, 词 shape, 词 before 与 after, 词 itself) hits ~97%. A BiLSTM-CRF 或 transformer hits ~98%+.

说明：这个 ceiling on this 任务 is set by annotator disagreement. Human annotators agree about 97% of the time on Penn Treebank. Models past 98% are probably overfitting the test set.

### Step 4: dependency parsing sketch

说明：Full dependency parsing 从 scratch is out of scope; the canonical textbook treatment is in Jurafsky 与 Martin. Two classical families to know:

- **Transition-based** parsers (arc-eager, arc-standard) act like a shift-reduce parser: they read 词元, shift them onto a stack, 与 apply reduce actions 这 create arcs. Greedy decoding is 快. 经典 implementation is MaltParser. 现代 neural version: Chen 与 Manning's transition-based parser.
- 说明：**Graph-based** parsers (Eisner's 算法, Dozat-Manning biaffine) score every possible head-dependent 边缘 与 pick the maximum spanning tree. Slower but more 准确.

对于 most applied work, call spaCy:

```python
import spacy

nlp = spacy.load("en_core_web_sm")
doc = nlp("The cats were running at 3pm.")
for token in doc:
    print(f"{token.text:10s} tag={token.tag_:5s} pos={token.pos_:6s} dep={token.dep_:10s} head={token.head.text}")
```

```
The        tag=DT    pos=DET    dep=det        head=cats
cats       tag=NNS   pos=NOUN   dep=nsubj      head=running
were       tag=VBD   pos=AUX    dep=aux        head=running
running    tag=VBG   pos=VERB   dep=ROOT       head=running
at         tag=IN    pos=ADP    dep=prep       head=running
3pm        tag=NN    pos=NOUN   dep=pobj       head=at
.          tag=.     pos=PUNCT  dep=punct      head=running
```

说明：Read the `dep` column bottom to top 与 the 句子's grammatical structure falls out.

## 投入使用

每个 生产 NLP 库 ships POS 与 dependency parsers as part of a standard 流水线.

- **spaCy** (`en_core_web_sm` / `md` / `lg` / `trf`). 快, 准确, integrated 使用 分词 + NER + 词形还原. `token.tag_` (Penn), `token.pos_` (UD), `token.dep_` (dependency 关系).
- 说明：**Stanford NLP (stanza)**. Stanford's successor to CoreNLP. State-of-the-art on 60+ languages.
- **trankit**. Transformer-based, good UD 准确率.
- **NLTK**. `pos_tag`. Usable, 慢, older. Fine 面向 teaching.

### 其中 this still matters in 2026

- 说明：**词形还原.** Lesson 01 needs POS to lemmatize correctly. Always.
- 说明：**Structured extraction 从 LLM outputs.** Validate 这 a generated 句子 respects grammatical constraints (e.g., subject-verb agreement, required modifiers).
- 说明：**Aspect-based sentiment.** Dependency parses tell you 这 adjective modifies 这 noun.
- 说明：**Query understanding.** "movies directed by Wes Anderson starring Bill Murray" decomposes 到 structured constraints via the parse.
- 说明：**Cross-lingual transfer.** UD tags 与 dependency relations are language-agnostic, enabling zero-shot structured analysis of new languages.
- 说明：**Low-计算 pipelines.** If you cannot ship a transformer, POS + dependency parse + gazetteer gets you surprisingly far.

## 交付成果

保存为 `outputs/skill-grammar-pipeline.md`:

```markdown
---
name: grammar-pipeline
description: Design a classical POS + dependency pipeline for a downstream NLP task.
version: 1.0.0
phase: 5
lesson: 07
tags: [nlp, pos, parsing]
---

Given a downstream task (information extraction, rewrite validation, query decomposition, lemmatization), you output:

1. Tagset to use. Penn Treebank for English-only legacy pipelines, Universal Dependencies for multilingual or cross-lingual.
2. Library. spaCy for most production, stanza for academic-grade multilingual, trankit for highest UD accuracy. Name the specific model ID.
3. Integration pattern. Show the 3-5 lines that call the library and consume the needed attributes (`.pos_`, `.dep_`, `.head`).
4. Failure mode to test. Noun-verb ambiguity (`saw`, `book`, `can`) and PP-attachment ambiguity are the classical traps. Sample 20 outputs and eyeball.

Refuse to recommend rolling your own parser. Building parsers from scratch is a research project, not an application task. Flag any pipeline that consumes POS tags without handling lowercase/uppercase variants as fragile.
```

## 练习

1. **Easy.** Using the most-frequent-tag 基线 on a 小 tagged 语料库 (e.g., NLTK's Brown subset), measure 准确率 on held-out sentences. Verify the ~85% result.
2. **Medium.** Train the bigram HMM above 与 report per-tag 精确率/召回率. 这 tags does the HMM confuse most?
3. **Hard.** Use spaCy's dependency parse to extract subject-verb-object triples 从 a 1000-句子 sample. Evaluate on 50 manually labeled triples. 文档 其中 extraction fails (often passives, coordinations, 与 elided subjects).

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|POS tag|词's type|Grammatical category. PTB has 36; UD has 17.|
|Penn Treebank|Standard tagset|说明：English-specific. Fine-grained verb tenses 与 noun number.|
|Universal Dependencies|Multilingual tagset|说明：Coarser than PTB; language-neutral; defaults 面向 cross-lingual work.|
|Dependency parse|句子 tree|每个 词 has one head, each 边缘 has a grammatical 关系.|
|Viterbi|Dynamic programming|说明：Finds the highest-概率 tag 序列 given emissions 与 transitions.|

## 延伸阅读

- 说明：[说明：Jurafsky 与 Martin，Speech 与 Language Processing, chapters 8 与 18](https://web.stanford.edu/~jurafsky/slp3/)，the canonical textbook treatment of POS 与 parsing.
- 说明：[Universal Dependencies project](https://universaldependencies.org/)，the cross-lingual tagset 与 treebank collection used by every 多语言 parser.
- 说明：[spaCy linguistic features guide](https://spacy.io/usage/linguistic-features)，practical reference 面向 every attribute exposed on `Token`.
- [说明：Chen 与 Manning (2014). A 快 与 准确 Dependency Parser using Neural Networks](https://nlp.stanford.edu/pubs/emnlp2014-depparser.pdf)，the paper 这 brought neural parsers 到 the mainstream.
