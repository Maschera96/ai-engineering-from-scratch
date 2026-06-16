---
name: attention-variant-picker-zh
description: 中文版, Pick a full / sliding-window / sparse / differential attention topology for a new model given context length, retrieval demands, and compute profile.
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
---

# Attention Variant Picker 中文版

请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。

## Inputs to gather

请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。

## Decision rules

- **Context ≤ 16K and retrieval ≤ 3**: full attention with FlashAttention. Don't optimize prematurely.
- **Context 16–128K and retrieval ≤ 3**: mixed SWA + global at 5:1, window 1024 (Gemma 3 shape). Keeps retrieval workable while collapsing KV.
- **Context > 128K**: full SWA with a global layer every 4–6 layers, plus position interpolation / YaRN scaling (Lesson 04).
- **Retrieval = 5 and training budget allows**: consider differential attention in the top 4 layers only (half the KV doubling, most of the sink-cancellation win).
- **You're shipping a public API**: prefer stable patterns (full, SWA, Gemma-3 mix). Skip native-sparse / DIFF unless you have kernel engineers.
- **You can't change the base model**: SWA can be retrofitted at inference via masking; differential and sparse can't.

## Always flag

- Pure-SWA models below 7B often lose measurably on reasoning benchmarks. Recommend against.
- Window size < 512 is almost never right. Go bigger or use a different topology.
- Differential attention reports in the paper are on small models (3–7B). Scale-up evidence is thin as of early 2026.
- Every variant interacts with RoPE / YaRN scaling (Lesson 04). State the position scheme explicitly.

## Output format

请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。

请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
请用简体中文执行本 artifact 的说明。保留代码、命令、路径、URL、标识符和 API 名称；翻译面向用户的解释、问题、判断标准和输出说明。
