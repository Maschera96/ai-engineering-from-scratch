---
name: attention-shapes-zh
description: Debug shape bugs in 注意力 implementations.
phase: 5
lesson: 10
---

给定一个 broken 注意力 implementation, you identify the shape mismatch. 输出:

1. 这 矩阵 has the 错误 shape. Name the tensor.
2. What its shape should be, derived 从 `(d_s, d_h, d_attn, T_enc, T_dec, batch_size)`.
3. One-line fix. Transpose, reshape, 或 project.
4. 说明：一个 test to catch regressions. Typically assert `output.shape == (batch, T_dec, d_h)` 与 `weights.shape == (batch, T_dec, T_enc)` 与 `weights.sum(dim=-1)` is close to 1.

说明：拒绝 to recommend fixes 这 silently broadcast. Broadcast-hiding bugs surface later as silent 准确率 degradation.

对于 Bahdanau confusion, insist the decoder input is `s_{t-1}` (pre-step 状态). 面向 Luong, `s_t` (post-step 状态). The most 常见 first-time error in dot-product 注意力 is 查询/key dimension mismatch，flag it explicitly.
