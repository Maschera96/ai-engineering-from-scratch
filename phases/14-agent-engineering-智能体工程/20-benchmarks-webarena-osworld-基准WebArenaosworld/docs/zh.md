# 基准：WebArena 与 OSWorld

> WebArena 在四个自托管应用上测试 Web 智能体的能力。OSWorld 在 Ubuntu、Windows、macOS 上测试桌面智能体的能力。发布时（2023–2024），两者都显示出顶尖智能体与人类之间存在巨大差距。差距正在缩小；但失败模式没有改变。

**类型：** 学习
**语言：** Python（标准库）
**前置要求：** Phase 14 · 19（SWE-bench、GAIA）
**时间：** 约 60 分钟

## 学习目标

- 描述 WebArena 的四个自托管应用，以及为什么基于执行的评估很重要。
- 解释 OSWorld 为什么使用真实操作系统截图，而不是无障碍 API。
- 说出 OSWorld 的两种主要失败模式：GUI grounding 与操作知识。
- 总结 OSWorld-G 和 OSWorld-Human 在基础基准之上增加了什么。

## 问题所在

通用智能体能够调用工具。但它们能驱动浏览器、跨越 20 次点击完成一次购物结账吗？它们能仅用键盘和鼠标配置一台 Linux 机器吗？这些正是 WebArena 和 OSWorld 要回答的问题。

## 核心概念

### WebArena（Zhou et al., ICLR 2024）

- 812 个长周期任务，分布在四个自托管 Web 应用上：一个购物网站、一个论坛、一个类 GitLab 的开发工具、一个商业 CMS。
- 外加一些实用工具：地图、计算器、便签。
- 评估基于执行，通过 gym API 完成——订单是否下单成功、issue 是否关闭、CMS 页面是否更新？
- 发布时：最强的 GPT-4 智能体成功率为 14.41%，而人类为 78.24%。

自托管这一设定很重要——由于目标应用被固定且可复现，基准不会出现不稳定的情况。

### 扩展

- **VisualWebArena** —— 视觉接地（visually grounded）任务，成功与否取决于对图像的解读（截图作为一等观察）。
- **TheAgentCompany**（2024 年 12 月）—— 增加终端 + 编码；更像真实的远程工作环境。

### OSWorld（Xie et al., NeurIPS 2024）

- 369 个真实计算机任务，跨越 Ubuntu、Windows、macOS。
- 对真实应用进行自由形式的键盘与鼠标控制。
- 以 1920×1080 截图作为观察。
- 发布时：最强模型 12.24%，而人类为 72.36%。

### 主要失败模式

1. **GUI grounding。** 像素 → 元素的映射。模型难以在 1920×1080 中可靠地定位 UI 元素。
2. **操作知识。** 设置在哪个菜单里、用哪个键盘快捷键、在哪个偏好设置面板。这是人类经年累月积累的长尾知识。

### 后续工作

- **OSWorld-G** —— 564 个样本的 grounding 套件 + Jedi 训练集。将 grounding 与规划解耦，从而可以分别度量它们。
- **OSWorld-Human** —— 人工整理的黄金动作轨迹。表明顶尖智能体使用的步数比必要步数多 1.4-2.7 倍（即轨迹效率差距）。

### 为什么这很重要

Claude computer use、OpenAI CUA、Gemini 2.5 Computer Use（第 21 课）全都基于由 WebArena 和 OSWorld 塑造的工作负载进行训练。基准是目标；生产模型则是交付的答案。

### 基准测试容易出错的地方

- **仅截图评估。** OSWorld 是截图驱动的；在 OSWorld 上评估一个使用 DOM 或无障碍 API 的智能体，会绕过 grounding 这一挑战。
- **忽视轨迹长度。** 只对成功率打分，会错过 OSWorld-Human 揭示的 1.4-2.7 倍步数低效问题。
- **过时的自托管应用。** WebArena 的应用固定了特定版本；在未重新整理的情况下升级会破坏可比性。

## 动手构建

`code/main.py` 实现了一个玩具级 Web 智能体测试框架：

- 一个最小化的“购物应用”状态机：list_items、add_to_cart、checkout。
- 针对 3 个任务的黄金轨迹。
- 一个尝试完成每个任务的脚本化智能体。
- 基于执行的评估器（状态检查）以及轨迹效率指标（步数 vs 黄金轨迹）。

运行它：

```
python3 code/main.py
```

输出：每个任务的成功率和轨迹效率，与 OSWorld-Human 的方法论保持一致。

## 实际应用

- **WebArena Verified** 自托管在内部集群上，用于持续评估。
- **OSWorld** 部署在虚拟机集群中，用于桌面智能体。
- **Computer-use 智能体**（第 21 课）—— Claude、OpenAI CUA、Gemini —— 全都基于类似这样的工作负载训练。
- **你自己的产品流程** —— 为你最重要的 20 个任务捕获黄金轨迹；每周让智能体对照它们运行。

## 交付物

`outputs/skill-web-desktop-harness.md` 构建一个 Web/桌面智能体测试框架，带有基于执行的评估和轨迹效率指标。

## 练习

1. 用第二个应用（一个论坛）扩展这个玩具测试框架。编写 3 个任务以及黄金轨迹。
2. 为每个任务添加轨迹效率报告。在你的玩具中，智能体是黄金轨迹的 1 倍、2 倍还是 3 倍？
3. 实现一个“干扰项”工具——一个黄金轨迹从不使用的工具。脚本化智能体会被诱惑吗？
4. 阅读 OSWorld-G。在你自己的评估中，你会如何将 grounding 失败与规划失败区分开？
5. 阅读 WebArena 的应用 README。当你升级某个固定版本的应用时，会有什么被破坏？

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| WebArena | “Web 智能体基准” | 812 个任务，分布在 4 个自托管应用上；gym 风格的评估 |
| VisualWebArena | “视觉 WebArena” | 视觉接地的 WebArena；截图作为观察 |
| OSWorld | “桌面智能体基准” | 369 个任务，运行在真实的 Ubuntu/Windows/macOS 上 |
| GUI grounding | “像素到元素的映射” | 模型在 1920x1080 中定位 UI 元素 |
| 操作知识 | “操作系统门道” | 哪个菜单、哪个快捷键、哪个偏好设置面板 |
| OSWorld-G | “grounding 套件” | 564 个纯 grounding 样本 + 训练集 |
| OSWorld-Human | “黄金轨迹” | 人工专家动作序列，用于度量效率 |
| 轨迹效率 | “相对黄金轨迹的步数” | 智能体步数除以人类最小步数 |

## 延伸阅读

- [Zhou et al., WebArena (arXiv:2307.13854)](https://arxiv.org/abs/2307.13854) —— 四应用 Web 基准
- [Xie et al., OSWorld (arXiv:2404.07972)](https://arxiv.org/abs/2404.07972) —— 跨操作系统的桌面基准
- [Anthropic, Introducing computer use](https://www.anthropic.com/news/3-5-models-and-computer-use) —— Claude 由基准塑造的能力
- [OpenAI, Computer-Using Agent](https://openai.com/index/computer-using-agent/) —— OSWorld 和 WebArena 数据
