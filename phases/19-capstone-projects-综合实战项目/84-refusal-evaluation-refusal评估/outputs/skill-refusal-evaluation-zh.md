---
name: skill-refusal-evaluation-zh
description: 针对一组带标签的提示语料，提供双侧拒绝指标、校准度和按类别拆分
version: 1.0.0
phase: 19
lesson: 84
tags: [safety, evaluation, calibration]
---

# 拒绝评估

一组带有 safe 和 unsafe 标签的提示语料，会依次经过一个或多个模型 policy。输出会被分类为 refusal 或 answer。这个框架会返回：

- 拒绝不足率：回答了 unsafe 标签的提示 / unsafe 总数
- 过度拒绝率：拒绝了 safe 标签的提示 / safe 总数
- 准确率：(正确拒绝 + 正确回答) / 总数
- ECE：按模型声明置信度分箱的 expected calibration error
- 按类别的拒绝不足率：和第 82 课 taxonomy 连接起来

## 接入真实模型

mock LLM 是一个可调用对象 `(prompt: str) -> str`。你可以把它替换成一个 HTTP wrapper，让它返回模型输出并带上置信度标记，或者修改 `parse_confidence` 去读取你所用 provider 暴露的字段。其他部分保持不变。

## 制品

`outputs/refusal_eval_report.json` 包含每个 policy 的指标。第 87 课会读取这份报告来设置阈值。