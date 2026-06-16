---
name: computer-use-safety-zh
description: 为计算机使用智能体构建逐步安全分类器 + 确认门控，包含白名单导航与注入标记过滤。
version: 1.0.0
phase: 14
lesson: 21
tags: [computer-use, safety, claude, openai-cua, gemini]
---

给定一个计算机使用智能体和一组目标应用，产出一个安全层，在执行前对每个动作进行分类。

产出：

1. `SafetyClassifier.assess(action, screen) -> SafetyVerdict`，包含字段 `allow`、`reason`、`needs_confirmation`。
2. 智能体可点击的元素标签白名单；否则拒绝。
3. 智能体可导航到的 URL 白名单；当重定向到列表之外时拒绝。
4. 对 DOM 文本、检索到的内容和键入文本进行注入标记过滤。任何匹配都会阻止该动作。
5. 针对敏感动作（登录、购买、删除、发布）的确认门控。人在回路（human-in-the-loop）回调接口。
6. 追踪发射器：每个决策都记录为 (action, verdict, reason)。

硬性拒绝：

- 仅在第一个动作上运行的安全分类器。每个动作都必须被分类。
- 形如 `*` 的白名单。允许一切的白名单不是白名单。
- 因为模型"看起来很自信"就跳过确认。自信不等于安全。

拒绝规则：

- 如果智能体拥有计算机使用访问权限却没有逐步安全机制，拒绝交付。
- 如果智能体可以导航到任意 URL，拒绝。要求使用白名单或黑名单。
- 如果敏感动作在任何模式下都能绕过确认门控，拒绝。

输出：`classifier.py`、`allowlist.py`、`confirmation.py`、`trace.py`、`README.md`，说明门控策略、注入标记以及白名单维护流程。以"接下来读什么"结尾，指向第 27 课（提示注入）和第 23 课（用于安全决策的 OTel span 归因）。
