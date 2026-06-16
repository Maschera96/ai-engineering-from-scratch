# A2A — 代理到代理协议
> Google 于 2025 年 4 月宣布 A2A；到 2026 年 4 月，该规范已在 https://a2a-protocol.org/latest/specification/ 上发布，并获得 150 多个组织的支持。 A2A 是 MCP（第 13 课）的水平补充：其中 MCP 是垂直的（代理 ↔ 工具），A2A 是点对点（代理 ↔ 代理）。它定义了代理卡（发现）、具有工件的任务（文本、结构化数据、视频）、不透明的任务生命周期和身份验证。生产系统越来越多地将 MCP 与 A2A 配对。 Google Cloud 在 2025 年至 2026 年期间将 A2A 支持纳入 Vertex AI Agent Builder。
**类型：** 学习 + 构建
**语言：** Python（stdlib、`http.server`、`json`）
**先决条件：** 第 16 阶段·04（原始模型）
**时间：** ~75 分钟
＃＃ 问题
您的代理需要呼叫另一个系统上的另一个代理。如何？您可以公开 HTTP 端点，定义定制的 JSON 模式，并希望对方说出它。每对代理都成为一个自定义集成。
A2A 是该呼叫的通用有线协议。标准发现、标准任务模型、标准传输、标准工件。类似于 HTTP+REST，但对于代理来说是一等公民。
＃＃ 概念
### 四个要素
**代理卡。** `/.well-known/agent.json` 上的 JSON 文档，描述代理：名称、技能、端点、支持的模式、身份验证要求。通过读卡来发现。
```
GET https://agent.example.com/.well-known/agent.json
→ {
    "name": "code-review-agent",
    "skills": ["review-python", "review-typescript"],
    "endpoints": {
      "tasks": "https://agent.example.com/tasks"
    },
    "auth": {"type": "bearer"},
    "modalities": ["text", "structured"]
  }
```

**任务。** 工作单元。具有生命周期的异步、有状态对象：`submitted → working → completed / failed / canceled`。客户端发送任务、轮询或订阅更新。
**工件。** 任务产生的结果类型。文本、结构化 JSON、图像、视频、音频。工件被分类，因此不同的方式都是一流的。
**不透明的生命周期。** A2A 没有规定远程代理“如何”解决任务。客户端看到状态转换和工件；实现可以免费使用任何框架。
### MCP/A2A 分裂
- **MCP**（第 13 课）：代理 ↔ 工具。代理reads/writes通过JSON-RPC连接到工具服务器。默认无状态。
- **A2A**：代理 ↔ 代理。对等协议；双方都是代理人，各有各的道理。
生产多代理系统同时使用这两种方法。 A2A 对等方调用其一侧的 MCP 工具。这种分裂使这两个问题保持清晰。
### 发现流程
```
Client                     Agent server
  ├──GET /.well-known/agent.json──>
  <──Agent Card JSON─────────────
  ├──POST /tasks {skill, input}──>
  <──201 task_id, state=submitted
  ├──GET /tasks/{id}──────────────>
  <──state=working, 42% done──────
  ├──GET /tasks/{id}──────────────>
  <──state=completed, artifacts──
```

或者使用流式传输：SSE 订阅 `/tasks/{id}/events` 以进行推送更新。
### 授权
A2A 支持三种常见模式：
- **不记名令牌** — OAuth2 或不透明。
- **mTLS** — 相互 TLS；组织相互证明身份。
- **签名请求** — 通过有效负载的 HMAC。
授权在代理卡中声明；客户发现并遵守。
### 到 2026 年 4 月将有 150 多个组织
企业采用推动了 A2A 的规模。标题：A2A 成为企业代理系统跨越信任边界的方式。 Google Cloud 提供了 Vertex AI Agent Builder A2A 支持；微软代理框架支持它；大多数主要框架（LangGraph、CrewAI、AutoGen）都附带 A2A 适配器。
### A2A 获胜的地方
- **跨组织呼叫。** A 公司的代理呼叫 B 公司的代理。如果没有 A2A，每对都是定制合同。
- **异构框架。** LangGraph 代理调用 CrewAI 代理调用自定义 Python 代理。 A2A 标准化。
- **键入的工件。** 视频结果、结构化 JSON、音频 — 都是一流的。
- **长时间运行的任务。** 不透明的生命周期+轮询使长达数小时的任务变得简单。
### A2A 的困境
- **延迟敏感的微调用。** A2A 的生命周期是异步的。亚毫秒级代理到代理不适合；使用直接 RPC。
- **紧密耦合的进程内代理。** 如果两个代理在同一个 Python 进程中运行，则 A2A 的 HTTP 往返就太过分了。
- **小团队。** 规格开销是真实存在的；仅限内部代理可能不需要手续。
### A2A 与 ACP、ANP、NLIP
2024-2026 年出现了几个相关规范：
- **ACP**（IBM/Linux 基金会）——A2A 的前身，范围更窄。
- **ANP**（代理网络协议）——对等发现重，去中心化优先。
- **NLIP**（Ecma 自然语言交互协议，2025 年 12 月标准化）——自然语言内容类型。
截至 2026 年 4 月，A2A 是采用最多的对等协议。有关比较，请参阅 arXiv:2505.02279（Liu 等人，“代理互操作性协议调查”）。
## 构建它
`code/main.py` 使用 `http.server` 和 JSON 实现 A2A 最小服务器和客户端。服务器：
- 公开 `/.well-known/agent.json`，
- 接受`POST /tasks`，
- 管理任务状态，
- 返回 `GET /tasks/{id}` 上的工件。
客户：
- 获取代理卡，
- 提交任务，
- 民意调查直至完成，
- 读取工件。
跑步：
```
python3 code/main.py
```

该脚本在后台线程中启动服务器，然后针对它运行客户端。您会看到完整的流程：发现、提交、轮询、工件。
## 使用它
`outputs/skill-a2a-integrator.md` 设计了 ​​A2A 集成：代理卡内容、任务模式、身份验证选择、流式传输与轮询。
## 发货
清单：
- **固定规范版本。** A2A 仍在不断发展；代理卡应声明协议版本。
- **幂等任务创建。** 重复提交（网络重试）应生成一项任务。
- **工件模式。** 声明代理返回的内容；消费者应该验证。
- **速率限制 + 授权。** A2A 面向公众；应用标准网络安全。
- **失败任务的死信。** 随着时间的推移检查重复出现的故障类型的模式。
## 练习
1.运行`code/main.py`。确认客户端发现服务器并收到正确的工件。
2.向服务器添加第二个技能（例如“总结”）。更新代理卡。编写一个根据任务类型选择技能的客户端。
3. 实现 SSE 流端点：发出状态更改的 `/tasks/{id}/events`。客户需要做什么不同的事情？
4. 阅读 A2A 规范 (https://a2a-protocol.org/latest/specification/)。确定本演示未实现的规范要求的三件事。
5. 将 A2A（代理卡发现）与 MCP（通过 `listTools` 列出服务器端功能）进行比较。自描述代理和能力探测之间的权衡是什么？
## 关键术语
|术语 |人们怎么说|它实际上意味着什么 ||------|----------------|------------------------|
| A2A | “代理对代理”|代理跨系统调用其他代理的对等协议。谷歌 2025。
|代理卡| 《代理商的名片》| `/.well-known/agent.json` 处的 JSON 描述技能、端点、身份验证。 |
|任务| “工作单位”|具有生命周期的异步有状态对象；完成时产生的工件。 |
|神器| “结果”|类型化输出：文本、结构化 JSON、图像、视频、音频。一流媒体。 |
|不透明的生命周期 | “怎么解决是代理商的事”|客户端看到状态转换；服务器可自由选择framework/tools. |
|发现 | “寻找代理”| `GET /.well-known/agent.json` 返回卡。 |
| MCP 与 A2A | “工具与同行”| MCP：垂直代理↔工具。 A2A：水平代理↔代理。 |
| ACP / ANP / NLIP | “兄弟协议”|相关规格； A2A 是 2026 年采用最多的。
## 进一步阅读
- [A2A 规范](https://a2a-protocol.org/latest/specification/) — 规范规范
- [Google 开发者博客 — A2A 公告](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/) — 2025 年 4 月发布帖子
- [A2A GitHub 存储库](https://github.com/a2aproject/A2A) — 参考实现和 SDK
- [刘等人。 — 代理互操作性协议调查](https://arxiv.org/html/2505.02279v1) — MCP、ACP、A2A、ANP 比较