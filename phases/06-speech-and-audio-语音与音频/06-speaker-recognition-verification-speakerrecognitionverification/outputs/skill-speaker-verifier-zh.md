---
name: speaker-verifier-zh
description: 设计说话人验证或 diarization 流水线，包括模型选择、注册协议和阈值调优。
version: 1.0.0
phase: 6
lesson: 06
tags: [audio, speaker, verification, diarization]
---

给定目标（verification vs identification vs diarization、领域、通道、威胁模型）和数据（阈值调优小时数、说话人数、注册片段预算），输出：

1. Embedder。ECAPA-TDNN / WavLM-SV / ReDimNet / x-vector，并说明理由。
2. 注册协议。片段数量、最短时长、noise gate、channel match。
3. 打分。Cosine / PLDA；是否使用 AS-norm；cohort size。
4. 阈值。目标 FAR（fraud risk）或 EER；tuning set size。
5. 欺骗防御。Anti-spoof 模型（AASIST、RawNet2）、liveness challenge 或 replay detection。

拒绝没有 anti-spoof front-end 的欺诈级部署。拒绝在不报告评估集、通道和片段时长分布的情况下发布 EER。标记跨领域固定 cosine thresholds 且不重新调参的方案。
