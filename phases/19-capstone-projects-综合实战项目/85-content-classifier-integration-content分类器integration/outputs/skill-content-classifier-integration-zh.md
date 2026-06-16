---
name: skill-content-classifier-integration-zh
description: 三个输出侧分类器（toxicity、PII、instruction-leakage）挂在同一个严重度路由器后面，支持 block、redact、warn、log 动作
version: 1.0.0
phase: 19
lesson: 85
tags: [safety, classifier, output-filter]
---

# 内容分类器集成

三个分类器，一个 router，四个动作。

## verdict 结构

```text
ClassifierVerdict
  name: str
  severity: none | low | medium | high
  score: float in [0, 1]
  findings: list[str]
```

## 动作表

| 严重度 | 动作 | 效果 |
|---|---|---|
| high | block | 输出被 policy refusal 替换 |
| medium | redact | 按分类器依次应用 redactor |
| low | warn | 原样发送输出，但附上一条软提示 |
| none | log | 原样发送输出，只记录 verdict |

## 各分类器行为

- toxicity，检测侮辱和骚扰词，要求有空白边界，并做一个小的左侧窗口否定检查，redact 成 `[redacted-language]`
- pii，检测 email、电话、SSN、通过 Luhn 校验的卡号、IPv4；SSN 和卡号会提升严重度；每种形状都替换成标签
- instruction-leakage，把输出和已知 system prompt 做三元组余弦比较；重叠越高，严重度越高；会移除第一行 system-prompt 标头

## 制品

`outputs/classifier_report.json` 保存每个样例的动作动词、严重度、redacted 后的输出，以及完整的 verdict 列表。