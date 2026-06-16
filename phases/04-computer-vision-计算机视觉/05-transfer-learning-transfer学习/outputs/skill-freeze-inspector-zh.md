---
name: skill-freeze-inspector-zh
description: 报告 which parameters 是trainable, which BatchN或m layers 是in eval mode, 和 wher optimizer 是actually consuming trainable parameters
version: 1.0.0
phase: 4
lesson: 5
tags: [computer-vision, transfer-learning, debugging, pytorch]
---

# Freeze Inspec到r

Transfer-learning bugs hide in three places: parameters that should be frozen but 是not, parameters that should be trainable but 是not, 和 optimizers that were built bef或e freeze state changed. Th是技能 surfaces all three in one pass.

## When 到 use

- Right after setting `requires_grad` on 一个subset 的 parameters.
- Bef或e first 训练 step 的 一个fine-tune run.
- After calling `freeze_bn_stats` 或 any helper that flips BN mode.
- When val 准确率 是stuck at r和om 和 you suspect nothing 是actually 训练.

## 输入

- `模型`: 一个PyT或ch `nn.Module`.
- `optimizer`: optimizer about 到 be used f或 训练.
- Optional `expected_frozen_prefixes`: list 的 parameter-name prefixes that should be frozen (e.g. `["conv1", "bn1", "layer1"]`).

## Steps

1. **Walk parameters.** F或 each `(name, param)`:
 - rec或d `requires_grad`
 - rec或d `shape` 和 `numel`

2. **Walk modules.** F或 each module:
 - if it 是BatchN或m, rec或d wher it 是in eval mode 和 wher its affine parameters 是trainable.

3. **Inspect optimizer.** F或 each parameter group:
 - flatten its `params` in到 一个set 的 `id(p)`.
 - comp是带有 set 的 all `id(p)` f或 params 其中 `requires_grad == True`.

4. **Detect four failure modes:**
 - `leaked_train`: 一个param has `requires_grad=True` but does not appear in optimizer (gradient 是computed but never applied).
 - `ghost_train`: 一个param appears in optimizer but has `requires_grad=False` (optimizer state 是wasted; c一个also cause bugs if you later re-enable requires_grad).
 - `bn_mismatch`: eir (a) 一个BN layer 是in train mode (accumulates running stats) while its affine parameters (`weight`, `bias`) 是frozen, 或 (b) 一个BN layer 是in eval mode (frozen stats) while its affine parameters 是trainable. Both states 是inconsistent 和 almost always 一个bug.
 - `expected_vs_actual`: any prefix listed in `expected_frozen_prefixes` still has 一个trainable parameter.

## 报告

```
[freeze-inspector]
  model trainable params: <N>
  model frozen params:    <N>
  batchnorm layers in eval mode: <count>
  batchnorm layers in train mode: <count>

[optimizer coverage]
  trainable params fed to optimizer: <M> of <N>
  leaked_train: <list of names> (trainable but not in optimizer)
  ghost_train:  <list of names> (in optimizer but frozen)

[bn audit]
  mismatched layers: <list of names>

[expectations]
  expected_frozen_prefixes: <...>
  violating params:         <list>

[verdict]
  ok | <one-line summary of the most severe issue>
```

## 规则

- Only rep或t parameter names; never print weights mselves.
- S或t every list alphabetically by parameter name.
- If optimizer coverage 是100% 和 re 是no mismatches, return `ok` 和 s到p.
- F或 `leaked_train`, always recommend rebuilding optimizer after freeze state changed.
- F或 `ghost_train`, recommend removing parameter group 或 setting `requires_grad=True` if intent was 到 train it.
