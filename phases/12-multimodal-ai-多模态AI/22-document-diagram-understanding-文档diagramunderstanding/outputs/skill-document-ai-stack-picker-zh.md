---
name: document-ai-stack-picker-zh
description: 根据领域、规模和合规需求，在 OCR 流水线、OCR-free 专用模型和 VLM 原生方案之间为文档 AI 项目做选型。
version: 1.0.0
phase: 12
lesson: 22
tags: [document-ai, ocr, donut, nougat, paligemma, vlm-native]
---

给定一个文档 AI 项目（领域：发票 / 科学论文 / 表单 / 混合；规模：每天处理的页数；质量标准；合规需求），选择一套技术栈并产出一份参考配置。

产出：

1. 技术栈选型。第 1 代（OCR 流水线 + LayoutLMv3）、第 2 代（Donut / Nougat OCR-free）、第 3 代（VLM 原生），或混合方案。
2. 单页成本估算。所选技术栈下的 token 数量与延迟。
3. 准确率预期。DocVQA + ChartQA + 领域专用基准。
4. 手写体策略。对成本不敏感时用 VLM 原生；面向规模时用专用 TrOCR + 路由。
5. 数学 / LaTeX 输出。科学论文用 Nougat；其他用 VLM。
6. 合规回退方案。带交叉核对审计日志的混合方案。

硬性拒绝项：
- 在没有成本分析的情况下，为 >100 万页/天的场景提议 VLM 原生方案。每页 2576px 的 token 成本相当可观。
- 为受监管的工作流推荐缺乏审计路径的单模型方案。
- 宣称 Nougat 能处理扫描发票。它不能——它是科学论文的专用模型。

拒绝规则：
- 如果规模 >1000 万页/天，拒绝第 3 代方案，推荐第 1 代，并将第 3 代作为抽样校验器。
- 如果领域以手写体为主，拒绝 OCR 流水线，推荐 VLM 原生 + 手写体专用模型（TrOCR）。
- 如果方程需要 LaTeX 保真度，则必须在流程中引入 Nougat。

输出：一页计划，包含技术栈、成本、准确率、手写体、数学、合规。结尾附上 arXiv 2308.13418（Nougat）、2204.08387（LayoutLMv3）、2111.15664（Donut）。
