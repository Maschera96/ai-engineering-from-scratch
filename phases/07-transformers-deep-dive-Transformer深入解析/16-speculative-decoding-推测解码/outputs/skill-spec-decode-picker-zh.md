---
name: spec-decode-picker-zh
description: 中文版, Pick a speculative decoding strategy (vanilla / Medusa / EAGLE / lookahead) and tuning parameters for a new LLM inference workload.
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
---

# Speculative Decoding Picker 中文版

请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。

## Inputs to gather

请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。

## Decision rules

- **Quick start, no training**: vanilla draft with a same-family 1B–3B model. 2× typical.
- **You can fine-tune**: EAGLE-2 or EAGLE-3 using the verifier's hidden states. 3–4× typical.
- **You can fine-tune but can't run two models**: Medusa (extra heads on verifier). 2–3×.
- **No training budget, no draft model available**: lookahead decoding. 1.3–1.6×.
- **Batch-heavy serving**: continuous batching matters more; speculative gains diminish as batch grows because the verifier is already saturated.
- **High temperature or stochastic sampling**: acceptance drops sharply. Consider lower N (2–3) or disabling.
- **Structured output (JSON, code)**: acceptance is high. Push N to 7+ for max speedup.

## Tuning

- **N (draft length)**: start at 5. Measure acceptance. If α > 0.9, push to 7. If α < 0.6, drop to 3.
- **Draft temperature**: match the verifier's temperature. Mismatched draft sampling loses α.
- **Tree depth (EAGLE-2 / Medusa)**: 3–5 branches; wider trees help only at α > 0.8.
- **Draft model size**: smallest that hits α > 0.7. A 1B draft for a 70B verifier is typical; don't go below the verifier's tokenizer / embedding compatibility.

## Always flag

- Check that draft and verifier share the tokenizer. Different BPE splits break speculative guarantees.
- Spec decoding interacts with continuous batching in vLLM: per-request speedup drops when the batch is already saturated.
- EAGLE's hidden-state input requires verifier internals; not always exposed through HF APIs. Prefer vLLM or SGLang runtimes.
- Medusa heads need a supervised fine-tune on the verifier's own outputs. Data-gathering step is often the dominant cost.

## Output format

请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。

请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
