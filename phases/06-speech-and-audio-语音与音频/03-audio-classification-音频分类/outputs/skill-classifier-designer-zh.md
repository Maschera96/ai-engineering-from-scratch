---
name: classifier-designer-zh
description: 为音频分类任务选择架构、增强、类别均衡策略和评估指标。
version: 1.0.0
phase: 6
lesson: 03
tags: [audio, classification, beats, ast]
---

给定音频分类任务（领域、标签数、每个片段的标签密度、数据量、部署目标），输出：

1. 架构。k-NN-MFCC / 2D CNN / AST / BEATs / Whisper-encoder，并给出一句理由。
2. 增强。SpecAugment 参数（time mask、freq mask 数量）、mixup α、背景噪声混合级别。
3. 类别均衡。Balanced sampler vs focal loss vs class weights，并绑定 tail-to-head ratio。
4. 损失与指标。CE / BCE / focal；主指标（top-1 / mAP / macro-F1）和次指标。
5. 切分与评估计划。Stratified k-fold；若是语音则 speaker-disjoint；若是流式数据则 temporal split。

拒绝只用 top-1 accuracy 评估多标签任务，必须使用 mAP。拒绝在没有 speaker-disjoint split 的情况下评估说话人相关任务。标记在少于 10k 标注片段上从零训练架构的方案，应从 SSL-pretrained backbone 开始。
