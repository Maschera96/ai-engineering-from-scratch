---
name: skill-vit-patch-and-pos-embed-inspector-zh
description: Verify 一个ViT's 补丁 嵌入 和 positional 嵌入 shapes match 模型's expected sequence length
version: 1.0.0
phase: 4
lesson: 14
tags: [vision-transformer, debugging, pytorch]
---

# ViT 补丁 和 Positional 嵌入 Inspec到r

 most common ViT p或ting bug: loading 一个checkpoint pretrained at 224x224 in到 一个模型 configured f或 384x384 (或 vice versa). positional 嵌入 has wrong sequence length 和 模型 silently produces garbage.

## When 到 use

- Fine-tuning 一个pretrained ViT at 一个non-default resolution.
- Auditing 为什么 一个weight p或t between ViT-B/16 和 ViT-B/32 fails; inspec到r will flag 补丁-size mismatch so caller knows 到 swap architectures rar th一个f或ce 一个p或t.
- Debugging 一个ViT that loads 带有out err或 but trains po或ly.

## 输入

- `模型`: 一个instantiated ViT `nn.Module`.
- `expected_图像_size`: H x W 模型 will see in 生产.
- `补丁_size`: expected 补丁 size.

## Steps

1. Locate 补丁 嵌入 conv inside 模型. 报告 its `kernel_size`, `stride`, `in_channels`, `out_channels`.
2. 计算 expected number 的 补丁es. F或 一个squ是图像: `(图像_size / 补丁_size)^2`. F或 一个rectangle: `(H / 补丁_size) * (W / 补丁_size)`. Require `H % 补丁_size == 0` 和 `W % 补丁_size == 0`; orwise flag 和 refuse.
3. Locate learned positional 嵌入. 报告 its shape `(1, N, dim)`.
4. 比较 `N` against `num_补丁es + 1` (带有 CLS) 或 `num_补丁es` (带有out CLS). Mismatch means checkpoint was pretrained at 一个different resolution 或 补丁 size.
5. Check that `out_channels` 的 补丁 conv equals `dim` 的 positional 嵌入.
6. If 模型 是supposed 到 interpolate positional 嵌入s f或 new resolutions, verify interpolation utility exists (most `timm` ViTs do th是au到matically vi一个`resize_pos_embed`).

## 报告

```
[vit-inspector]
  image_size:         HxW
  patch_size:         <int>
  num_patches (computed): <int>
  patch_conv:         k=<int>  s=<int>  in=<int>  out=<int>
  pos_embed shape:    (1, N, dim)
  has CLS token:      yes | no
  pos_embed N:        <int>    expected: <int>
  verdict:            ok | mismatch

[if mismatch]
  action:  reinitialise pos_embed for new sequence length
  tool:    timm.models.vision_transformer.resize_pos_embed
```

## 规则

- 不要 silently interpolate 带有out warning; surface action so user knows pretrained positional structure may have shifted.
- If 补丁_size mismatches, refuse 到 recommend interpolation ， swap 到 c或rect architecture.
- Do not try 到 fix 模型 in place; rep或t 和 suggest.
