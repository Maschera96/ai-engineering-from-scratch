---
name: ai-scientist-zh
description: 构建一个自主研究代理，运行实验树搜索，通过视觉批判撰写 LaTeX 论文，并通过沙箱逃逸红队。
version: 1.0.0
phase: 19
lesson: 05
tags: [capstone, autonomous-agent, ai-scientist, sakana, langgraph, sandbox, research]
---

给定一个种子想法、一个狭窄的领域和 30 美元的计算预算，构建一个运行实验树搜索的代理，撰写可审查的 LaTeX 论文，并发出可重复性包。
建设计划：
1.文献通行证：Semantic Sc​​holar Graph API + OpenAlex；在 FAISS 中缓存摘要；生成 1 页的域摘要。
2. 树搜索：使用 <span class="notranslate">`expand(node) -> children`</span>（每个子项一次配置编辑）和 <span class="notranslate">`score(node) = novelty*0.4 + quality*0.5 + budget*0.1`</span> 在实验节点上实现最佳优先扩展。
3. 每节点沙箱：每个实验运行 <span class="notranslate">`docker run --network=none --memory=8g --cpus=2 --pids-limit=256 --read-only`</span> 或 E2B 等效项；确定性种子；强制执行资源上限。
4. 计划-执行-验证：验证步骤检查损失收敛、基线运行、消融隔离索赔。
5. 编写器：生成 LaTeX，编译为 PDF，将 PDF 提供给 Claude Opus 4.7 视觉模式，用于对布局和声明证据对齐进行批判，最多迭代 3 次。
6. 审稿人整体：五名评委（Opus 4.7、GPT-5.4、Gemini 3 Pro、DeepSeek R1、Qwen3-Max）对 NeurIPS 评分标准（新颖性、严谨性、清晰度、可重复性、影响力）进行评分；意思是 < 4.0 returns to writer.
7. Red team: integrate adversarial tasks (fork bomb, filesystem escape, LLM-written network call). Confirm all blocked. Emit <span class="notranslate">`red_team.md`</span>。
8. 可重复性捆绑包：paper.pdf + review.md + 树搜索跟踪 JSON + 种子 + W&B 运行链接 + 沙箱配置 + 一行重新运行命令。
评估标准：
|重量 |标准|测量|
|:-:|---|---|
| 25 | 25纸张质量 |针对同一种子主题已发表的研讨会论文进行盲审
| 20 |实验严谨|基线、种子、消融；每个声明都由结果表中的单元格支持|
| 20 |成本和计算规则|每张纸强制执行 30 美元上限，Langfuse 追踪 |
| 20 |安全|沙盒红队通行证；网络策略和终止开关已通过记录的尝试进行验证
| 15 | 15再现性|一个命令重新运行即可用相同的种子重现论文 |
硬拒绝：
- 在沙箱外运行的实验。顶峰的整个论点是执行是受到限制的。
- 编写步骤不重新阅读已编译的 PDF（视觉批判是承重的）。
- 没有基线、种子或消融部分的论文。
- 成本预算仅作为事后警告执行，而不是硬性上限。
拒绝规则：
- 拒绝发表审稿人意思低于 4.0/5 且没有明确的人工覆盖的论文。
- 拒绝运行需要从沙箱内部访问网络的种子想法。相反，添加一个单独的只读数据集卷。
- 拒绝重刊其红队尚未被执行和记录的论文。
输出：一个包含树搜索引擎、沙箱策略、writer/reviewer 循环的存储库、三个使用再现性捆绑包运行的示例、一份红队报告、一份成本分类帐 csv 以及一份说明您再现了哪些 Sakana v2 故障模式以及缓解措施如何工作的文章。