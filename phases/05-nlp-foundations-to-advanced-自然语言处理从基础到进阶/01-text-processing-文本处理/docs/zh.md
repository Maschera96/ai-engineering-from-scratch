# 文本处理，分词、词干提取、词形还原

> 语言是连续的，模型是离散的，预处理连接两者。

**类型:** 构建
**语言:** Python
**先修要求:** Phase 2 · 14 (Naive Bayes)
**时间:** 约45分钟

## 问题

说明：一个 模型 cannot read "The cats were running." It reads integers.

每个 NLP 系统 opens 使用 the same three questions. 其中 does a 词 start. What is the root of the 词. How do we treat "run", "running", "ran" as the same thing when it helps, 与 as different things when it doesn't.

Get 分词 错误 与 the 模型 learns 从 garbage. If your 分词器 treats `don't` as one 词元 but `do n't` as two, the 训练 分布 splits. If your stemmer collapses `organization` 与 `organ` to the same stem, 主题建模 dies. If your lemmatizer needs part-of-speech context but you don't pass it, verbs get treated as nouns.

说明：本课 builds the three preprocessing steps 从 scratch, then shows how NLTK 与 spaCy do the same work so you can see the tradeoffs.

## 概念

Three operations. Each has a job 与 a 失败模式.

**分词** splits a string 到 词元. "词元" is deliberately vague 因为 the right granularity depends on the 任务. 词-level 面向 classical NLP. Subword 面向 transformers. Character 面向 languages 不使用 whitespace.

**词干提取** chops suffixes 使用 rules. 快, aggressive, dumb. `running -> run`. `organization -> organ`. 这 second one is the 失败模式.

**词形还原** reduces a 词 to its dictionary form using grammar knowledge. Slower, 准确, needs a lookup table 或 morphological analyzer. `ran -> run` (needs to know "ran" is past tense of "run"). `better -> good` (needs to know comparative forms).

Rule of thumb. Stem when speed matters 与 you can tolerate 噪声 (search indexing, rough 分类). Lemmatize when meaning matters (问题 answering, 语义 search, anything the 用户 will read).

```figure
edit-distance
```

## 动手构建

### Step 1: a regex 词 分词器

说明：这个 simplest useful 分词器 splits on non-alphanumeric characters while keeping punctuation as its own 词元. Not perfect, not final, but it runs in one line.

```python
import re

def tokenize(text):
    return re.findall(r"[A-Za-z]+(?:'[A-Za-z]+)?|[0-9]+|[^\sA-Za-z0-9]", text)
```

说明：Three patterns in order of precedence. 词 使用 optional inner apostrophe (`don't`, `it's`). Pure numbers. Any single non-whitespace non-alphanumeric character as a standalone 词元 (punctuation).

```python
>>> tokenize("The cats weren't running at 3pm.")
['The', 'cats', "weren't", 'running', 'at', '3', 'pm', '.']
```

Failure modes to notice. `3pm` splits to `['3', 'pm']` 因为 we alternated between letter runs 与 digit runs. Good enough 面向 most tasks. URLs, emails, hashtags all break. 面向 生产, add patterns before the general ones.

### Step 2: a Porter stemmer (step 1a only)

说明：这个 full Porter 算法 has five phases of rules. Step 1a alone covers the most frequent English suffixes 与 teaches the pattern.

```python
def stem_step_1a(word):
    if word.endswith("sses"):
        return word[:-2]
    if word.endswith("ies"):
        return word[:-2]
    if word.endswith("ss"):
        return word
    if word.endswith("s") and len(word) > 1:
        return word[:-1]
    return word
```

```python
>>> [stem_step_1a(w) for w in ["caresses", "ponies", "caress", "cats"]]
['caress', 'poni', 'caress', 'cat']
```

说明：Read the rules top-down. The `ies -> i` rule is why `ponies -> poni`, not `pony`. Real Porter has step 1b 这 would fix it. Rules compete. Earlier rules win. The order matters more than any single rule.

### Step 3: a lookup-based lemmatizer

说明：词形还原 proper needs morphology. A tractable teaching version uses a 小 lemma table 与 a fallback.

```python
LEMMA_TABLE = {
    ("running", "VERB"): "run",
    ("ran", "VERB"): "run",
    ("runs", "VERB"): "run",
    ("better", "ADJ"): "good",
    ("best", "ADJ"): "good",
    ("cats", "NOUN"): "cat",
    ("cat", "NOUN"): "cat",
    ("were", "VERB"): "be",
    ("was", "VERB"): "be",
    ("is", "VERB"): "be",
}

def lemmatize(word, pos):
    key = (word.lower(), pos)
    if key in LEMMA_TABLE:
        return LEMMA_TABLE[key]
    if pos == "VERB" and word.endswith("ing"):
        return word[:-3]
    if pos == "NOUN" and word.endswith("s"):
        return word[:-1]
    return word.lower()
```

```python
>>> lemmatize("running", "VERB")
'run'
>>> lemmatize("cats", "NOUN")
'cat'
>>> lemmatize("better", "ADJ")
'good'
>>> lemmatize("watched", "VERB")
'watched'
```

这个 last case is the key teaching moment. `watched` is not in our table 与 our fallback only handles `ing`. Real 词形还原 covers `ed`, irregular verbs, comparative adjectives, plurals 使用 sound changes (`children -> child`). This is why 生产 systems use WordNet, spaCy's morphologizer, 或 a full morphological analyzer.

### Step 4: pipe them together

```python
def preprocess(text, pos_tagger=None):
    tokens = tokenize(text)
    stems = [stem_step_1a(t.lower()) for t in tokens]
    tags = pos_tagger(tokens) if pos_tagger else [(t, "NOUN") for t in tokens]
    lemmas = [lemmatize(word, pos) for word, pos in tags]
    return {"tokens": tokens, "stems": stems, "lemmas": lemmas}
```

说明：这个 missing piece is a POS tagger. Phase 5 · 07 (POS Tagging) builds one. 面向 now, 默认 everything to `NOUN` 与 acknowledge the limitation.

## 投入使用

说明：NLTK 与 spaCy ship the 生产 versions. A few lines each.

### NLTK

```python
import nltk
nltk.download("punkt_tab")
nltk.download("wordnet")
nltk.download("averaged_perceptron_tagger_eng")

from nltk.tokenize import word_tokenize
from nltk.stem import PorterStemmer, WordNetLemmatizer
from nltk import pos_tag

text = "The cats were running."
tokens = word_tokenize(text)
stems = [PorterStemmer().stem(t) for t in tokens]
lemmatizer = WordNetLemmatizer()
tagged = pos_tag(tokens)


def nltk_pos_to_wordnet(tag):
    if tag.startswith("V"):
        return "v"
    if tag.startswith("J"):
        return "a"
    if tag.startswith("R"):
        return "r"
    return "n"


lemmas = [lemmatizer.lemmatize(t, nltk_pos_to_wordnet(tag)) for t, tag in tagged]
```

说明：`word_tokenize` handles contractions, Unicode, 边缘 cases your regex misses. `PorterStemmer` runs all five phases. `WordNetLemmatizer` needs the POS tag translated 从 NLTK's Penn Treebank scheme to WordNet's abbreviation set. The 翻译 wiring above is the bit most tutorials skip.

### spaCy

```python
import spacy

nlp = spacy.load("en_core_web_sm")
doc = nlp("The cats were running.")

for token in doc:
    print(token.text, token.lemma_, token.pos_)
```

```
The      the     DET
cats     cat     NOUN
were     be      AUX
running  run     VERB
.        .       PUNCT
```

spaCy hides the whole 流水线 behind `nlp(text)`. 分词, POS tagging, 与 词形还原 all run. Faster than NLTK at scale. More 准确 out of the box. The tradeoff is 这 you cannot easily swap individual components.

### 如何选择

|Situation|Pick|
|-----------|------|
|Teaching, research, swapping components|NLTK|
|Production, multi-language, speed matters|spaCy|
|Transformer 流水线 (you'll tokenize 使用 the 模型's 分词器 anyway)|Use `tokenizers` / `transformers` 与 skip classical preprocessing|

### 这个 two failure modes nobody warns you about

大多数 tutorials teach the algorithms 与 stop. Two things will bite a real preprocessing 流水线, 与 they are almost never covered.

**Reproducibility drift.** NLTK 与 spaCy change 分词 与 lemmatizer behavior between versions. What produced `['do', "n't"]` in spaCy 2.x may produce `["don't"]` in 3.x. Your 模型 was trained on one 分布. 推理 now runs on a different one. 准确率 quietly degrades 与 nobody knows why. Pin 库 versions in `requirements.txt`. Write a preprocessing regression test 这 freezes expected 分词 of 20 sample sentences. Run it on every upgrade.

**训练 / 推理 mismatch.** Train 使用 aggressive preprocessing (lowercase, stopword removal, 词干提取), deploy on 原始 用户 input, watch performance crater. This is the single most 常见 生产 NLP failure. If you preprocess during 训练, you must run the identical 函数 during 推理. Ship preprocessing as a 函数 inside the 模型 package, not as a notebook cell the serving team rewrites.

## 交付成果

说明：一个 reusable prompt 这 helps engineers pick a preprocessing strategy 不使用 reading three textbooks.

保存为 `outputs/prompt-preprocessing-advisor.md`:

```markdown
---
name: preprocessing-advisor
description: Recommends a tokenization, stemming, and lemmatization setup for an NLP task.
phase: 5
lesson: 01
---

You advise on classical NLP preprocessing. Given a task description, you output:

1. Tokenization choice (regex, NLTK word_tokenize, spaCy, or transformer tokenizer). Explain why.
2. Whether to stem, lemmatize, both, or neither. Explain why.
3. Specific library calls. Name the functions. Quote the POS-tag translation if NLTK is involved.
4. One failure mode the user should test for.

Refuse to recommend stemming for user-visible text. Refuse to recommend lemmatization without POS tags. Flag non-English input as needing a different pipeline.
```

## 练习

1. 说明：**Easy.** Extend `tokenize` to keep URLs as single 词元. Test: `tokenize("Visit https://example.com today.")` should produce one URL 词元.
2. 说明：**Medium.** Implement Porter step 1b. If a 词 contains a vowel 与 ends in `ed` 或 `ing`, remove it. Handle the double-consonant rule (`hopping -> hop`, not `hopp`).
3. **Hard.** Build a lemmatizer 这 uses WordNet as a lookup table but falls back to your Porter stemmer when WordNet has no entry. Measure 准确率 on a tagged 语料库 against plain WordNet 与 plain Porter.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|词元|一个 词|说明：Whatever unit the 模型 consumes. Can be 词, subword, character, 或 byte.|
|Stem|Root of a 词|说明：Result of rule-based suffix stripping. Not always a real 词.|
|Lemma|Dictionary form|说明：这个 form you'd look up. Requires grammatical context to 计算 correctly.|
|POS tag|Part of speech|说明：Category like NOUN, VERB, ADJ. Needed to lemmatize accurately.|
|Morphology|词 shape rules|说明：How a 词 changes form based on tense, number, case. 词形还原 depends on it.|

## 延伸阅读

- 说明：[Porter, M. F. (1980). An 算法 面向 suffix stripping](https://tartarus.org/martin/PorterStemmer/def.txt)，the original paper, five pages, still the clearest explanation.
- 说明：[spaCy 101，linguistic features](https://spacy.io/usage/linguistic-features)，how a real 流水线 is wired.
- 说明：[NLTK book, chapter 3](https://www.nltk.org/book/ch03.html)，分词 边缘 cases you haven't thought of yet.
