---
name: skill-prompt-injection-detector-zh
description: 分层检测器管线，输入任意提示都能返回类别和置信度，并且 precision 和 recall 可度量
version: 1.0.0
phase: 19
lesson: 83
tags: [safety, detector, prompt-injection]
---

# 提示注入检测器

这里的 detector 是一个从 prompt 到 verdict 的函数。verdict 带有第 82 课 taxonomy 里的某个类别，以及 `[0, 1]` 之间的置信度。

## 管线

1. 归一化，去掉零宽字符，纠正同形异码，解码 base64/hex，把 leet 语的数字折回字母，并尝试 rot13，再用常见词做一次 sanity check。
2. 子串规则，手写一些短语，例如 `ignore previous`、`from now on you are`、`decode this base64`。
3. 正则规则，词元级模式，例如 `ignor\w*\s+(all|prior|previous|earlier)`。

聚合器会保留每个类别的最高分，并返回得分最大的类别。如果没有任何规则触发，就返回 `benign`。

## 新增规则

编辑 `code/rules.py`。一条规则就是一个字典，包含 `name`、`category`（六类 taxonomy 之一）、`score`（0 到 1 的浮点数），以及 `substring` 或 `regex` 其中之一。重新运行 `main.py`，查看它对按类别 precision 和 recall 的影响。

## 制品

`outputs/detector_report.json` 是按类别的指标文件。第 87 课的端到端 gate 会读取它，用来设置置信度阈值。