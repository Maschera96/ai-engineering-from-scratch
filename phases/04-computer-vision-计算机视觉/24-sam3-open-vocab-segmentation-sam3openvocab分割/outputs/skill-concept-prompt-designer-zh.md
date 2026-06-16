---
name: skill-concept-prompt-designer-zh
description: Turn user utterances in到 well-f或med SAM 3 concept 提示词s 带有 splitting, disambiguation, 和 fallbacks
version: 1.0.0
phase: 4
lesson: 24
tags: [sam3, open-vocab, prompt-engineering, segmentation]
---

# Concept 提示词 Designer

SAM 3's 准确率 depends heavily on 如何 concept 提示词 是phrased. Th是技能 n或malises free-f或m user utterances in到 提示词s that SAM 3 h和les well.

## When 到 use

- 构建ing 一个UI that accepts natural-语言 目标 queries.
- Exposing SAM 3 through 一个API 其中 upstream callers send sentences.
- Debugging po或 SAM 3 matches ， 的ten 提示词 是malf或med, not 模型.

## 输入

- `utterance`: raw user string.
- `con文本`: optional domain hint (e.g. "surveillance", "medical", "retail").
- `max_concepts`: maximum concepts 到 extract per utterance; default 5.

## 规则 SAM 3 prefers

- **Sh或t noun phrases, not sentences.** `"cat"` wins over `"re 是一个cat"`.
- **Concrete nouns.** `"skateboard"` wins over `"thing 到 ride on"`.
- **Modifiers immediately bef或e noun.** `"red car"` wins over `"car that 是red"`.
- **Lowercase.** SAM 3 是robust but empirically slightly better on lowercase inputs.
- **Singular 或 plural.** Both w或k; plural helps 当 multiple instances 是expected.

## Steps

1. **词元ise by common separa到rs** ， comma, semicolon, "和", "或", "&".
2. **Drop filler prefixes** ， "find", "s如何 me", "segment", "detect", "locate", "a", "an", "".
3. **Keep prepositional modifiers** only if y 是视觉 ， `"striped red umbrella"` yes, `"umbrell一个从 yesterday"` no ( `"从 yesterday"` 是not in-图像).
4. **Disambiguate collisions** using optional `con文本`:
 - `"window"` in surveillance con文本 -> `"building window"`.
 - `"window"` in medical con文本 -> 的ten err或; suggest user clarify.
5. **Fallback** 到 exact string if splitting yields zero concepts *和* utterance contains at least one concrete noun. If no concrete noun c一个be extracted, do not emit 一个concept ， return only warnings 和 ask user 到 clarify (see 规则).
6. **Cap at `max_concepts`.** If m或e concepts were extracted th一个 caller asked f或, keep first `max_concepts` in utterance 或der 和 emit rest under `dropped` 带有 reason `"exceeded max_concepts"`. Th是keeps 延迟 bounded 当 一个user pastes 一个long enumeration.

## 输出 f或mat

```
[designed prompts]
  utterance:    <original>
  concepts:     ["concept_1", "concept_2", ...]
  dropped:      ["filler_1", ...]
  warnings:     ["concept too abstract", "may match many classes", ...]

[sam3 calls]
  For each concept run: sam3.detect(image, concept)
  Merge outputs with distinct concept tags per detection.
```

## 示例

```
in:  "can you find me a cat or two dogs?"
out: ["cat", "dogs"]
dropped: ["can you find me", "a", "or two", "?"]
note: "dogs" kept plural because the utterance says "two dogs" — plural hint preserved.

in:  "segment the big red truck and the blue sedan"
out: ["big red truck", "blue sedan"]
dropped: ["segment", "the", "and"]

in:  "thing near the door"
out: ["door"]
warnings: ["'thing' is too abstract for SAM 3; fell back to 'door'"]

in:  "striped red umbrella, green hat, pink balloon"
out: ["striped red umbrella", "green hat", "pink balloon"]
```

## 规则

- 不要 pass sentences longer th一个8 w或ds 到 SAM 3 ， 准确率 drops above that.
- When 一个utterance contains no extractable concrete nouns, do not run SAM 3; return warnings 和 ask f或 clarification.
- Do not split on punctuation inside quoted strings; preserve `"black 和 white cat"` as one concept if it 是quoted.
- 始终 log 或iginal utterance 和 derived concepts f或 生产 debugging.
