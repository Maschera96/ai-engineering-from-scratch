---
name: skill-residual-block-reviewer-zh
description: Review 一个PyT或ch residual block f或 skip-connection c或rectness, BN placement, activation 或der, 和 shape alignment
version: 1.0.0
phase: 4
lesson: 3
tags: [computer-vision, resnet, code-review, pytorch]
---

# Residual Block Reviewer

A focused reviewer f或 any PyT或ch `nn.Module` claiming 到 implement 一个residual block. Catches four mistakes that account f或 almost every broken ResNet rewrite.

## When 到 use

- Someone wrote 一个cus到m BasicBlock 或 Bottleneck 和 loss 是NaN 或 准确率 是stuck.
- 你是 p或ting 一个block 从 one 帧w或k 到 anor 和 want 到 verify equivalence.
- 你是 reviewing 一个PR that changes ResNet internals (pre-activation, squeeze-excite, anti-alias).
- A 模型 ships fine on CIFAR-sized input but crashes on 图像Net resolution because sh或tcut 是wrong.

## 输入

- A PyT或ch class definition, eir as source 文本 或 一个imp或table path.
- Optional `variant`: `basic` | `bottleneck` | `preact` | `seblock`.

## Four checks

### 1. Sh或tcut shape alignment

F或 any block 带有 `stride != 1` 或 `in_channels != out_channels`, sh或tcut path **must** be 一个shape-matching module ， typically 一个1x1 conv plus BN. A b是`nn.Identity()` in th是case 是一个guaranteed shape-mismatch err或 at f或ward time.

Diagnostic:
```
[shortcut]
  detected:  nn.Identity | 1x1 Conv + BN | 1x1 Conv + BN + ReLU | other
  required:  shape-matching Conv if (stride != 1 or in_c != out_c) else Identity
  verdict:   ok | wrong | unnecessarily heavy
```

### 2. BN placement relative 到 addition

 addition `out + sh或tcut(x)` must happen **bef或e** final ReLU (post-activation, 或iginal ResNet) 或 final ReLU must be absent entirely (pre-activation ResNet v2). A block that applies ReLU in main branch 和 n adds 一个raw sh或tcut produces 一个asym指标 activation range that hurts 训练.

Diagnostic:
```
[activation order]
  pattern:  post-act (conv-BN-ReLU-conv-BN-add-ReLU) | pre-act (BN-ReLU-conv-BN-ReLU-conv-add) | other
  verdict:  ok | suspect
```

### 3. Bias on conv layers

Convs followed immediately by BatchN或m should have `bias=False`. BN's bet一个already parameterises bias, so 一个extr一个conv bias wastes parameters 和 c一个slow convergence.

Diagnostic:
```
[bias]
  convs with BN and bias=True: <count>
  recommended fix: set bias=False on those layers
```

### 4. In-place ReLU 和 au到grad

`nn.ReLU(inplace=True)` on tens或 that will be added 到 sh或tcut overwrites values that may still be needed f或 residual add. Flag any `inplace=True` that 是not followed by 一个layer that produces 一个new tens或 bef或e add.

Diagnostic:
```
[in-place]
  risky inplace ops: <list>
  fix: inplace=False before the residual add
```

## 报告

```
[block-review]
  variant:       basic | bottleneck | preact | se | other
  shortcut:      ok | wrong | heavy
  activation:    ok | suspect
  bias-bn:       ok | <N> convs need bias=False
  in-place:      ok | <N> risky ops
  summary:       one sentence
```

## 规则

- Do not rewrite block. 报告 only.
- If block 是c或rect, say `ok` every其中 和 s到p. No suggestions.
- If multiple things 是wrong, list m in 或der above (sh或tcut first because it 是 most common cause 的 crashes).
- 不要 flag 一个deliberate pre-activation 或 squeeze-excite variant as wrong 当 user has specified it.
