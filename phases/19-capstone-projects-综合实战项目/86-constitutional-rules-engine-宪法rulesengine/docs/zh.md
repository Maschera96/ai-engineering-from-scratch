# 结题 86：宪法规则引擎

> 规则由名称、谓词和解释组成。少了这三样里任何一样的东西，都只是感觉，不是规则。

**类型：** Build
**语言：** Python, YAML
**先修：** 第18阶段安全课程，第19阶段 A 轨第25到29课
**时间：** 约90分钟

## 问题

分类器覆盖的是那些一看就能认出来的失败。规则引擎覆盖的是契约性的失败。团队在写 coding assistant 时，可能需要一个约束：“每一条包含代码的回复，最后都必须是一个可运行的代码块，或者明确写出一个假设。”客服机器人团队可能需要：“每一次拒绝，都必须提供下一步动作。”这些约束不天然适合分类器。它们是对 response、对话以及系统策略的谓词，而且需要让非工程师也能读懂。

诚实的表达方式是一个声明式文件。constitution 以 YAML 的形式和代码并存，受版本控制管理，并且有单独的审查流程。每条规则都有 `name`、`predicate`、`severity` 和 `explanation` 模板。引擎会加载这个文件，针对候选输出逐条求值，并为触发的每条规则返回一个结构化 `Violation`。这个结题项目里的规则引擎把 `all_of`、`any_of` 和 `not_` 组合起来，这样一条规则就能表达“如果 response 包含代码，它必须以一个可运行块结尾，并且不能引用内部专用库”。

这节课的另一半是修订。一个只会拦截的规则引擎，只做了一半。一个还能建议修复的规则引擎，才真正能投入运营：助手先草拟回复，引擎标出违规，修复器产出修订版，然后引擎确认修订版满足规则。这个 lesson 附带一个最小修复器（按规则做正则替换）和一个结构化 diff（草稿与修订版之间逐行的新增、删除、编辑）。

## 概念

```mermaid
flowchart LR
  D[草稿回复] --> RE[规则引擎]
  RE -->|违规| F[修复器]
  F --> R[修订回复]
  R --> RE2[规则引擎第二遍]
  RE2 -->|结论| OUT[接受或升级处理]
  D -.->|diff| R
```

一条规则的形状如下：

```yaml
- name: end-with-runnable-or-assumption
  severity: medium
  applies_when:
    contains_regex: '```python'
  must:
    any_of:
      - ends_with_regex: '```\s*$'
      - contains_regex: 'assumption:'
  explanation: "代码回复必须以闭合代码围栏或明确的假设结尾。"
  fix:
    append_if_missing: "

Assumption: example inputs are valid."
```

谓词是原子的：`contains_regex`、`not_contains_regex`、`ends_with_regex`、`starts_with_regex`、`max_words`、`min_words`。组合方式是 `all_of`、`any_of`、`not_`。引擎会先计算 `applies_when`；如果规则不适用，就把 violation 记录成 `not_applicable`。否则引擎会计算 `must`，并产生 `pass` 或 `violation`。

严重度是 `low`、`medium`、`high`，和第 85 课保持一致。下游的 gate（第 87 课）会把 `high` 级规则违规，和 `high` 级分类器 verdict 一样对待：直接 block。

修复器是一组声明式操作：`append_if_missing`、`prepend_if_missing`、`replace_regex`。每个操作都把某条规则映射成一个变换。这个修复器刻意只做局部编辑；结构性重写属于一个独立的拒绝与帮助层，这里不展开。

diff 会基于原文和修订文计算。它是一组 `Change` 记录，带有 `op`（add、remove、edit）以及相关文本。下游 gate 可以把 diff 写进日志，方便人工审查修复器的行为演变。

## 构建

`code/rules.yml` 保存 constitution。`code/main.py` 里的 loader 同时支持 YAML 文件（如果装了 PyYAML）和 JSON 文件（内置）。这个 lesson 随附的 `rules.yml` 可以被 lesson tests 通过两条代码路径都解析。`code/main.py` 定义了 `Engine` 和 `Fixer` 类，以及一个 `diff` 函数。组合谓词会递归求值，并在 `any_of` 上短路。

本 lesson 里的 constitution 包含：

- `no-empty-refusal`（medium）- 拒绝必须包含建议或者转向
- `end-with-runnable-or-assumption`（medium）- 代码回复必须干净收尾
- `no-pii-in-examples`（high）- 示例数据不能包含 email 或电话形状
- `cite-when-asserting-fact`（low）- 以 “According to” 开头的行必须带括号里的引用
- `no-internal-library-leak`（high）- `internal-only` 和 `policybot-internal` 这两个词不能出现在输出里
- `bounded-length`（low）- 回复不能超过 800 个词

## 使用

运行 `python3 main.py`。演示会把 3 段草稿回复送进引擎，打印违规项，运行修复器，打印 diff，并写出 `outputs/rules_report.json`。其中一个样例里有一条不适用规则（草稿里没有代码块），报告会把这条规则标成 `not_applicable`，让团队明确看到引擎确实检查过它。

## 交付

`outputs/skill-constitutional-rules-engine.md` 记录了规则语法以及修复器操作。

## 练习

1. 新增一条规则：当提示提到 safety 时，每条回复都必须包含 “If this is urgent” 这句话。使用组合谓词来写。
2. 把正则修复器替换成一个带命名槽位的模板修复器。用一条规则演示新设计下的改写。
3. 新增一个 metrics endpoint。给定一批草稿，返回每条规则的违规率，这样团队就能看出哪些规则触发过头。

## 关键术语

| 术语 | 常见用法 | 精确定义 |
|---|---|---|
| constitution | 一个模糊的策略文档 | 一个由规则、谓词、严重度和解释组成的 YAML 文件 |
| predicate | 一个检查 | 一个从文本到 bool 的可调用对象，可以是原子的，也可以通过 all_of/any_of/not_ 组合 |
| violation | 一个失败 | 一条结构化记录，包含规则名、严重度、解释和匹配片段 |
| fixer | 一个模型微调 | 一个按规则确定性地把草稿变成修订版的变换 |
| diff | 一个字符串比较 | 一份描述草稿和修订版之间 add、remove、edit 操作的结构化列表 |

## 进一步阅读

第 87 课会把这个引擎和输入侧检测器、输出侧分类器组合成一个统一的安全 gate。