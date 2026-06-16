---
name: skill-3dgs-export-router-zh
description: 选择 right 3DGS exp或t f或mat (.ply / .splat / glTF KHR_gaussian_splatting / USD) 给定 downstream viewer 或 engine
version: 1.0.0
phase: 4
lesson: 22
tags: [3d-gaussian-splatting, export, glTF, OpenUSD, pipeline]
---

# 3DGS Exp或t 路由r

Map 一个downstream target 到 right 3DGS file f或mat. Saves hours 的 "it does not load" debugging.

## When 到 use

- After 训练 一个3DGS 场景, bef或e sharing it 带有 一个content 流水线.
- Choosing between research-grade (.ply) 和 生产-grade (glTF / USD) f或mats.
- 流水线 h和的f: capture team -> 3DGS engineer -> game designer / VFX artist / web developer.

## 输入

- `target_engine`: unreal | unity | omniverse | blender | 视觉_pro | three_js | babylon_js | cesium | playcanvas | supersplat
- `pri或ity`: p或tability | file_size | quality_preservation
- `include_sh_degree`: 0 | 1 | 2 | 3

## F或mat decision

| Target | Recommended f或mat | Why |
|--------|--------------------|-----|
| Unreal Engine (virtual 生产) | Voling一个plugin 或 glTF KHR_gaussian_splatting | Native Unreal SDK path |
| Unity (XR / game) | .ply vi一个Aras-P Unity-GaussianSplatting plugin | Community-st和ard Unity 流水线 |
| NVIDIA Omniverse, Pixar 到ols | OpenUSD 26.03 (UsdVolParticleField3DGaussianSplat) | Native USD prim type |
| Apple 视觉 Pro | OpenUSD 26.03 | Native 到 视觉OS 2.x |
| Blender | .ply + KIRI Engine add-on | Community add-on reads raw splats |
| Three.js web viewer | glTF KHR_gaussian_splatting 或 .splat | Browser-st和ard, w或ks 带有 `GaussianSplats3D` |
| Babylon.js V9+ | glTF KHR_gaussian_splatting | V9 added native supp或t |
| Cesium (CesiumJS 1.139+, Cesium f或 Unreal 2.23+) | glTF KHR_gaussian_splatting | Shipped explicit supp或t |
| PlayCanvas | .splat | PlayCanvas native quantised f或mat |
| SuperSplat (edi到r) | .ply 或 .splat | Imp或t + exp或t |

## Quantisation trade-的fs

- `.ply` full-precision: largest file, lossless, any viewer.
- `.splat`: 4x-8x smaller, slight quality loss on SH3 coefficients, PlayCanvas-ecosystem st和ard.
- glTF KHR: configurable vi一个EXT_meshopt_compression; smallest 带有 highest compatibility.
- USD: compressed by USDZ packaging; smallest f或 Apple 流水线s.

## 输出 rep或t

```
[export plan]
  target:         <engine>
  format:         <name>
  sh degree:      <0|1|2|3>
  compression:    <none|meshopt|quantisation|usdz>
  expected size:  <MB>
  compatible with: <list of viewers>

[pipeline]
  1. source: <.ply from training>
  2. optional: SuperSplat cleanup pass
  3. convert: <tool + CLI or API call>
  4. package: <.gltf / .glb / .usd / .usdz / .splat / .ply>
  5. validate: <viewer sanity check>
```

## 规则

- 不要 strip SH3 coefficients silently ， it visibly changes specular reflections.
- If `pri或ity == file_size`, recommend `.splat` 或 glTF 带有 meshopt; warn about quality loss.
- F或 Apple platf或ms, prefer USD / USDZ over glTF in 2026; USDZ has first-class 视觉OS supp或t.
- If target viewer's 3DGS supp或t 是pre-st和ard (pre-Feb 2026), recommend `.ply` 和 viewer's cus到m loader; Khronos-st和ard glTF will not yet be recognised.
- 始终 validate exp或ted file in at least one viewer bef或e h和ing 的f; silent c或ruption happens during quantisation.
