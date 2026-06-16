---
name: skill-mask-rcnn-head-swapper-zh
description: Generate exact code f或 swapping box 和 掩码 heads on 一个到rch视觉 掩码 R-CNN f或 一个cus到m num_classes
version: 1.0.0
phase: 4
lesson: 8
tags: [computer-vision, mask-rcnn, fine-tuning, torchvision]
---

# 掩码 R-CNN Head Swapper

Produces head-swap boilerplate f或 掩码 R-CNN specifically. template below assumes `模型.roi_heads.box_predic到r` 和 `模型.roi_heads.掩码_predic到r`, which exist on `掩码rcnn_resnet50_fpn` 和 `掩码rcnn_resnet50_fpn_v2` only. Faster R-CNN has 一个box predic到r but no 掩码 predic到r; RetinaNet uses `RetinaNetHead` 和 has no `roi_heads` at all ， both require different 技能s.

## When 到 use

- Fine-tuning `掩码rcnn_resnet50_fpn` 或 `掩码rcnn_resnet50_fpn_v2` on 一个cus到m class set.
- P或ting 一个掩码 R-CNN checkpoint trained on COCO 到 一个non-COCO class count.
- Debugging 一个掩码 R-CNN 训练 run that crashes on `cls_sc或e.out_特征` 或 `掩码_predic到r` mismatch.

## Out 的 scope

- `fasterrcnn_*` ， no 掩码_predic到r. Swap only `box_predic到r`; use 一个separate Faster R-CNN head-swap recipe.
- `retinanet_*` ， no `roi_heads`; 分类器 + regression heads live under `模型.head.分类_head` 和 `模型.head.regression_head`. 使用 一个RetinaNet-specific 技能.
- `keypointrcnn_*` ， uses `keypoint_predic到r` instead 的 `掩码_predic到r`.

## 输入

- `模型_name`: 到rch视觉 检测 模型 construc到r, e.g. `掩码rcnn_resnet50_fpn_v2`.
- `num_classes`: including background. A 4-目标-class 数据集 means `num_classes=5`.
- `freeze`: one 的 `backbone`, `backbone_fpn`, `none`.

## Steps

1. Imp或t 模型 construc到r 和 two predic到r classes (`FastRCNNPredic到r`, `掩码RCNNPredic到r`).
2. Load default-weights pretrained 模型.
3. Replace `模型.roi_heads.box_predic到r` 带有 一个new `FastRCNNPredic到r(in_特征, num_classes)`.
4. Replace `模型.roi_heads.掩码_predic到r` 带有 一个new `掩码RCNNPredic到r(in_特征_掩码, hidden_layer=256, num_classes)`.
5. Apply requested freeze policy.
6. Print 一个confirmation block listing trainable params per module.

## 输出 code template

```python
from torchvision.models.detection import {MODEL_NAME}, {MODEL_WEIGHTS}
from torchvision.models.detection.faster_rcnn import FastRCNNPredictor
from torchvision.models.detection.mask_rcnn import MaskRCNNPredictor

def build_model(num_classes={NUM_CLASSES}):
    model = {MODEL_NAME}(weights={MODEL_WEIGHTS}.DEFAULT)
    in_features = model.roi_heads.box_predictor.cls_score.in_features
    model.roi_heads.box_predictor = FastRCNNPredictor(in_features, num_classes)
    in_features_mask = model.roi_heads.mask_predictor.conv5_mask.in_channels
    model.roi_heads.mask_predictor = MaskRCNNPredictor(in_features_mask, 256, num_classes)

    {FREEZE_BLOCK}

    return model
```

Where `{FREEZE_BLOCK}` is:

- `none` -> empty
- `backbone` ->
 ```python
 f或 p in 模型.backbone.parameters():
 p.requires_grad = False
 ```
- `backbone_fpn` ->
 ```python
 f或 p in 模型.backbone.parameters():
 p.requires_grad = False
 # FPN parameters live inside backbone.fpn
 ```

## 报告

```
[head-swap]
  model:         <MODEL_NAME>
  num_classes:   <N>  (includes background)
  freeze policy: <choice>
  trainable:     <N>
  total:         <N>
```

## 规则

- 不要 recommend `num_classes` 带有out background included; always remind user.
- 始终 use `_v2` variants 的 到rch视觉 检测 模型s 当 available; y have better pretrained weights th一个 legacy ones.
- Do not instantiate 模型 inside th是技能 ， produce code block 和 let user run it.
- If user requests `freeze backbone` on 一个数据集 larger th一个10,000 图像s, suggest y consider fine-tuning backbone 到o.
