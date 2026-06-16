# 命名实体识别

> 说明：Pull the names out. Sounds easy until you deal 使用 ambiguous boundaries, nested entities, 与 领域 jargon.

**类型:** 构建
**语言:** Python
**先修要求:** Phase 5 · 02 (BoW + TF-IDF), Phase 5 · 03 (词 嵌入)
**时间:** 约75分钟

## 问题

"Apple sued Google over its iPhone search deal in the US." Five entities: Apple (ORG), Google (ORG), iPhone (PRODUCT), search deal (maybe), US (GPE). A good NER 系统 extracts all of them 使用 正确 types. A bad one misses iPhone, confuses Apple the fruit 使用 Apple the company, 与 labels "US" as a PERSON.

NER is the workhorse underneath every structured extraction 流水线. Resume parsing, compliance log scanning, medical record anonymization, search 查询 understanding, grounding 面向 聊天机器人 responses, legal contract extraction. You never quite see it; you always depend on it.

说明：本课 walks the classical path (rule-based, HMM, CRF) 到 the 现代 one (BiLSTM-CRF, then transformers). Each step solves a specific limitation of the one before it. The pattern is the lesson.

## 概念

**BIO tagging** (或 BILOU) turns 实体 extraction 到 a 序列-labeling problem. 标签 each 词元 使用 `B-TYPE` (beginning of 实体), `I-TYPE` (inside 实体), 或 `O` (outside any 实体).

```
Apple    B-ORG
sued     O
Google   B-ORG
over     O
its      O
iPhone   B-PRODUCT
search   O
deal     O
in       O
the      O
US       B-GPE
.        O
```

说明：Multi-词元 entities chain: `New B-GPE`, `York I-GPE`, `City I-GPE`. A 模型 这 understands BIO can extract arbitrary spans.

这个 architecture progression:

- 说明：**Rule-based.** Regex + gazetteer lookups. High 精确率 on known entities, zero coverage on new ones.
- **HMM.** Hidden Markov 模型. Emission 概率 of 词元 given tag, transition 概率 of tag-to-tag. Viterbi decode. Trained on labeled 数据.
- 说明：**CRF.** Conditional Random Field. Like HMM but discriminative, so you can mix arbitrary features (词 shape, capitalization, neighboring 词). Still the classical 生产 workhorse in 2026 面向 low-resource deployments.
- 说明：**BiLSTM-CRF.** Neural features instead of hand-crafted. LSTM reads the 句子 both directions, CRF layer on top enforces consistent tag sequences.
- **Transformer-based.** Fine-tune BERT 使用 a 词元-分类 head. Best 准确率. Most 计算.

```figure
ner-bio-tagging
```

## 动手构建

### Step 1: BIO tagging helpers

```python
def spans_to_bio(tokens, spans):
    labels = ["O"] * len(tokens)
    for start, end, label in spans:
        labels[start] = f"B-{label}"
        for i in range(start + 1, end):
            labels[i] = f"I-{label}"
    return labels


def bio_to_spans(tokens, labels):
    spans = []
    current = None
    for i, label in enumerate(labels):
        if label.startswith("B-"):
            if current:
                spans.append(current)
            current = (i, i + 1, label[2:])
        elif label.startswith("I-") and current and current[2] == label[2:]:
            current = (current[0], i + 1, current[2])
        else:
            if current:
                spans.append(current)
                current = None
    if current:
        spans.append(current)
    return spans
```

```python
>>> tokens = ["Apple", "sued", "Google", "over", "iPhone", "sales", "."]
>>> labels = ["B-ORG", "O", "B-ORG", "O", "B-PRODUCT", "O", "O"]
>>> bio_to_spans(tokens, labels)
[(0, 1, 'ORG'), (2, 3, 'ORG'), (4, 5, 'PRODUCT')]
```

### Step 2: hand-crafted features

说明：对于 classical (non-neural) NER, features are the game. Useful ones:

```python
def token_features(token, prev_token, next_token):
    return {
        "lower": token.lower(),
        "is_upper": token.isupper(),
        "is_title": token.istitle(),
        "has_digit": any(c.isdigit() for c in token),
        "suffix_3": token[-3:].lower(),
        "shape": word_shape(token),
        "prev_lower": prev_token.lower() if prev_token else "<BOS>",
        "next_lower": next_token.lower() if next_token else "<EOS>",
    }


def word_shape(word):
    out = []
    for c in word:
        if c.isupper():
            out.append("X")
        elif c.islower():
            out.append("x")
        elif c.isdigit():
            out.append("d")
        else:
            out.append(c)
    return "".join(out)
```

说明：`word_shape("iPhone")` returns `xXxxxx`. `word_shape("USA-2024")` returns `XXX-dddd`. Capitalization patterns are high-signal 面向 proper nouns.

### Step 3: a simple rule-based + dictionary 基线

```python
ORG_GAZETTEER = {"Apple", "Google", "Microsoft", "OpenAI", "Meta", "Amazon", "Netflix"}
GPE_GAZETTEER = {"US", "USA", "UK", "India", "Germany", "France"}
PRODUCT_GAZETTEER = {"iPhone", "Android", "Windows", "ChatGPT", "Claude"}


def rule_based_ner(tokens):
    labels = []
    for token in tokens:
        if token in ORG_GAZETTEER:
            labels.append("B-ORG")
        elif token in GPE_GAZETTEER:
            labels.append("B-GPE")
        elif token in PRODUCT_GAZETTEER:
            labels.append("B-PRODUCT")
        else:
            labels.append("O")
    return labels
```

说明：Production gazetteers have millions of entries scraped 从 Wikipedia 与 DBpedia. Coverage is good. Disambiguation (`Apple` the company vs the fruit) is terrible. 这 is why statistical models won.

### Step 4: the CRF step (sketch, not full impl)

说明：Full CRF 从 scratch in 50 lines is not enlightening 不使用 the 概率-theory foundations. Use `sklearn-crfsuite` instead:

```python
import sklearn_crfsuite

def to_features(tokens):
    out = []
    for i, tok in enumerate(tokens):
        prev = tokens[i - 1] if i > 0 else ""
        nxt = tokens[i + 1] if i + 1 < len(tokens) else ""
        out.append({
            "word.lower()": tok.lower(),
            "word.isupper()": tok.isupper(),
            "word.istitle()": tok.istitle(),
            "word.isdigit()": tok.isdigit(),
            "word.suffix3": tok[-3:].lower(),
            "word.shape": word_shape(tok),
            "prev.word.lower()": prev.lower(),
            "next.word.lower()": nxt.lower(),
            "BOS": i == 0,
            "EOS": i == len(tokens) - 1,
        })
    return out


crf = sklearn_crfsuite.CRF(algorithm="lbfgs", c1=0.1, c2=0.1, max_iterations=100, all_possible_transitions=True)
X_train = [to_features(s) for s in sentences_tokenized]
crf.fit(X_train, bio_labels_train)
```

`c1` 与 `c2` are L1 与 L2 regularization. `all_possible_transitions=True` lets the 模型 learn illegal sequences (e.g., `I-ORG` after `O`) are unlikely, 这 is how a CRF enforces BIO consistency 不使用 you writing the constraint.

### Step 5: what a BiLSTM-CRF adds

Features become learned. Inputs: 词元 嵌入 (GloVe 或 fastText). LSTM reads left-to-right 与 right-to-left. Concatenated hidden states go through a CRF 输出 layer. The CRF still enforces tag-序列 consistency; the LSTM replaces hand-crafted features 使用 learned ones.

```python
import torch
import torch.nn as nn


class BiLSTM_CRF_Head(nn.Module):
    def __init__(self, vocab_size, embed_dim, hidden_dim, n_labels):
        super().__init__()
        self.embed = nn.Embedding(vocab_size, embed_dim)
        self.lstm = nn.LSTM(embed_dim, hidden_dim, bidirectional=True, batch_first=True)
        self.fc = nn.Linear(hidden_dim * 2, n_labels)

    def forward(self, token_ids):
        e = self.embed(token_ids)
        h, _ = self.lstm(e)
        emissions = self.fc(h)
        return emissions
```

说明：对于 the CRF layer, use `torchcrf.CRF` (pip install pytorch-crf). The gain over hand-crafted CRF is measurable but smaller than you expect unless you have tens of thousands of labeled sentences.

## 投入使用

spaCy ships 生产-grade NER out of the box.

```python
import spacy

nlp = spacy.load("en_core_web_sm")
doc = nlp("Apple sued Google over its iPhone search deal in the US.")
for ent in doc.ents:
    print(f"{ent.text:20s} {ent.label_}")
```

```
Apple                ORG
Google               ORG
iPhone               ORG
US                   GPE
```

注意 `iPhone` labeled `ORG` rather than `PRODUCT`，spaCy's 小 模型 has weak product-实体 coverage. The 大 模型 (`en_core_web_lg`) does better. The transformer 模型 (`en_core_web_trf`) does better still.

Hugging Face 面向 BERT-based NER:

```python
from transformers import pipeline

ner = pipeline("ner", model="dslim/bert-base-NER", aggregation_strategy="simple")
print(ner("Apple sued Google over its iPhone in the US."))
```

```
[{'entity_group': 'ORG', 'word': 'Apple', ...},
 {'entity_group': 'ORG', 'word': 'Google', ...},
 {'entity_group': 'MISC', 'word': 'iPhone', ...},
 {'entity_group': 'LOC', 'word': 'US', ...}]
```

`aggregation_strategy="simple"` merges contiguous B-X, I-X 词元 到 a span. 不使用 it, you get 词元-level labels 与 have to merge yourself.

### LLM-based NER (the 2026 option)

说明：Zero-shot 与 few-shot LLM NER is now competitive 使用 fine-tuned models on many domains, 与 dramatically better when labeled 数据 is scarce.

- **Zero-shot prompting.** Give the LLM a list of 实体 types 与 an example 模式. Ask 面向 JSON 输出. Works out of the box; 准确率 is moderate on novel domains.
- **ZeroTuneBio-style prompting.** Decompose the 任务 到 candidate extraction → meaning explanation → judgment → re-check. A multi-stage prompt (not one-shot) lifts 准确率 substantially on biomedical NER. The same pattern works 面向 legal, financial, 与 scientific domains.
- **Dynamic prompting 使用 RAG.** Retrieve the most similar labeled examples 从 a 小 annotated seed set 面向 every 推理 call; build the few-shot prompt on the fly. In 2026 benchmarks, this lifts GPT-4 biomedical NER F1 by 11-12% over static prompting.
- **Per-实体-type decomposition.** 面向 长 文档, a single call 这 extracts all 实体 types at once loses 召回率 as length grows. Run one extraction pass per 实体 type. Higher 推理 成本, substantially higher 准确率. This is the standard pattern 面向 clinical notes 与 legal contracts.

Production recommendation as of 2026: start 使用 an LLM zero-shot 基线 before you collect 训练 数据. Often the F1 is good enough 这 you never need to fine-tune.

### 其中 classical NER still wins

说明：Even 使用 LLMs available, classical NER wins when:

- 延迟 预算 is under 50ms.
- 你有 thousands of labeled examples 与 need 98%+ F1.
- 说明：这个 领域 has a stable ontology 其中 a pretrained CRF 或 BiLSTM transfers well.
- 说明：Regulatory constraints require an on-prem, non-generative 模型.

### 其中 it falls apart

- 说明：**领域 shift.** CoNLL-trained NER on legal contracts performs worse than a gazetteer. Fine-tune on your 领域.
- 说明：**Nested entities.** "Bank of America Tower" is simultaneously an ORG 与 a FACILITY. Standard BIO cannot represent overlapping spans. You need nested NER (multi-pass 或 span-based models).
- 说明：**长 entities.** "United States Federal Deposit Insurance Corporation." 词元-level models sometimes split this. Use `aggregation_strategy` 或 post-process.
- 说明：**Sparse types.** Medical NER labels like DRUG_BRAND, ADVERSE_EVENT, DOSE. General-purpose models have no idea. Scispacy 与 BioBERT are the starting points there.

## 交付成果

保存为 `outputs/skill-ner-picker.md`:

```markdown
---
name: ner-picker
description: Pick the right NER approach for a given extraction task.
version: 1.0.0
phase: 5
lesson: 06
tags: [nlp, ner, extraction]
---

Given a task description (domain, label set, language, latency, data volume), output:

1. Approach. Rule-based + gazetteer, CRF, BiLSTM-CRF, or transformer fine-tune.
2. Starting model. Name it (spaCy model ID, Hugging Face checkpoint ID, or "custom, trained from scratch").
3. Labeling strategy. BIO, BILOU, or span-based. Justify in one sentence.
4. Evaluation. Use `seqeval`. Always report entity-level F1 (not token-level).

Refuse to recommend fine-tuning a transformer for under 500 labeled examples unless the user already has a pretrained domain model. Flag nested entities as needing span-based or multi-pass models. Require a gazetteer audit if the user mentions "production scale" and labels are unchanged from CoNLL-2003.
```

## 练习

1. 说明：**Easy.** Implement `bio_to_spans` (the inverse of `spans_to_bio`) 与 verify round-trip consistency on 10 sentences.
2. 说明：**Medium.** Train the sklearn-crfsuite CRF above on the CoNLL-2003 English NER dataset. Report per-实体 F1 using `seqeval`. Typical result: ~84 F1.
3. **Hard.** Fine-tune `distilbert-base-cased` on a 领域-specific NER dataset (medical, legal, 或 financial). Compare against the spaCy 小 模型. 文档 数据 leakage checks 与 write up what surprised you.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|NER|Extract names|标签 词元 spans 使用 types (PERSON, ORG, GPE, DATE, ...).|
|BIO|Tagging scheme|`B-X` begins, `I-X` continues, `O` outside.|
|BILOU|Better BIO|Adds `L-X` (last), `U-X` (unit) 面向 cleaner boundaries.|
|CRF|Structured classifier|说明：Models transitions between labels, not just emissions. Enforces 有效 sequences.|
|Nested NER|Overlapping entities|说明：One span is a different 实体 than a sub-span of it. BIO cannot express this.|
|Entity-level F1|Proper NER 指标|说明：Predicted span must match true span exactly. 词元-level F1 overstates 准确率.|

## 延伸阅读

- 说明：[说明：Lample et al. (2016). Neural Architectures 面向 Named Entity Recognition](https://arxiv.org/abs/1603.01360)，the BiLSTM-CRF paper. Canonical.
- [说明：Devlin et al. (2018). BERT: Pre-训练 of Deep Bidirectional Transformers](https://arxiv.org/abs/1810.04805)，introduces the 词元-分类 pattern 这 became standard.
- 说明：[说明：spaCy linguistic features，named entities](https://spacy.io/usage/linguistic-features#named-entities)，practical reference 面向 every attribute on `Doc.ents` 与 `Span`.
- [seqeval](https://github.com/chakki-works/seqeval)，the 正确 指标 库. Use it always.
