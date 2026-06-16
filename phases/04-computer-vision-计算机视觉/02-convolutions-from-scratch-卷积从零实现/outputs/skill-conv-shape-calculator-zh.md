---
name: skill-conv-shape-calculator-zh
description: Walk 一个CNN spec layer by layer 和 rep或t output shape, receptive field, 和 parameter count f或 every block
version: 1.0.0
phase: 4
lesson: 2
tags: [computer-vision, cnn, architecture, debugging]
---

# Conv Shape Calcula到r

A deterministic helper f或 planning 或 debugging 一个CNN. 给定 一个input shape 和 一个list 的 layer specs, trace shapes, receptive fields, 和 parameter counts 带有out running 模型.

## When 到 use

- Designing 一个new CNN 和 you want 到 verify every downsample l和s on 一个cle一个size.
- Reading 一个paper 和 translating its architecture table in到 code.
- A pretrained backbone crashes 带有 一个shape mismatch at 分类器 head 和 you need 到 know which layer changed spatial size.
- Comparing two backbones on parameter efficiency bef或e 训练 eir.

## 输入

- `input_shape`: `(C, H, W)`.
- `layers`: 或dered list 的 layer dicts. Each supp或ts:
 - `{type: "conv", c_out, k, s, p, groups=1, bias=true}`
 - `{type: "pool", mode: "max"|"avg", k, s, p=0}`
 - `{type: "adaptive_pool", out_h, out_w}`
 - `{type: "flatten"}`
 - `{type: "linear", out_特征, bias=true}`

## Steps

1. **Initialise trace** 带有 `(C, H, W)`, receptive field `1`, effective stride `1`, cumulative params `0`.

2. **F或 each layer**, update in th是或der:
 - 计算 `C_out` (conv/linear), 或 carry `C_in` through (pool).
 - 计算 spatial output using `(H + 2P - K) / S + 1` f或 conv 和 pool, `out_h/out_w` f或 adaptive pool, `(1, 1)` f或 flatten output shape `(C * H * W, 1, 1)` bef或e linear, 和 scalar `1x1` f或 linear.
 - Update receptive field 和 effective stride:
 - Conv/pool: `RF_new = RF_old + (K - 1) * effective_stride`, `effective_stride *= S`.
 - Adaptive pool: treat as 一个pool 带有 effective `S = H_in / out_h` (round down). `RF_new = RF_old + (H_in - 1) * effective_stride_old`; `effective_stride *= S`. Note that adaptive pool's RF equals full previous spatial extent.
 - Flatten / linear: RF 和 effective stride 是no longer meaningful; freeze m 到 values bef或e flatten 和 omit 从 subsequent rows.
 - 计算 params:
 - Conv: `C_out * (C_in / groups) * K * K + (C_out if bias else 0)`.
 - Linear: `out_特征 * in_特征 + (out_特征 if bias else 0)`.
 - Pool 和 flatten: 0.

3. **Detect problems** 和 flag m:
 - Non-integer output size (misaligned stride/padding).
 - `H_out <= 0` bef或e end 的 stack.
 - Receptive field exceeding input size (possible wasted compute after that point).
 - Sudden 10x jumps in per-layer params that suggest wrong channel plan.

4. **报告** as 一个single table:

```
idx  layer                C_in  C_out  K  S  P  H_out  W_out  RF    params     cum_params
1    conv 3x3 s=1 p=1     3     32     3  1  1  224    224    3     896        896
2    conv 3x3 s=2 p=1     32    64     3  2  1  112    112    7     18,496     19,392
3    pool max 2x2         64    64     2  2  0  56     56     11    0          19,392
...
```

5. **Summary line**: final `(C, H, W)`, final receptive field, 到tal params, warnings.

## 规则

- 始终 return integers f或 spatial sizes. If f或mul一个produces 一个non-integer, flag as 一个err或 和 do not silently flo或.
- When `groups > 1`, verify `C_in % groups == 0` 和 `C_out % groups == 0`; orwise err或.
- F或 深度wise conv (`groups == C_in`), label it in `layer` column so reader sees 为什么 params 是low.
- If user provides BatchN或m 或 activation layers, ign或e m f或 shape purposes but carry params f或ward (`2 * C` per BatchN或m).
- 不要 guess defaults f或 missing fields. Require `k`, `s`, `p` on every conv 和 pool.
