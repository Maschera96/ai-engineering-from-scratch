---
name: tripwire-design-zh
description: 审查智能体检测器栈，包括熔断开关、断路器和金丝雀，并标出缺失触发器。
版本: 1.0.0
phase: 15
lesson: 14
tags: [kill-switch, circuit-breaker, canary, honeytoken, detection-and-response]
---

给定a 拟议的 detector stack for an 智能体 部署, 审计 it 针对 the three-detector reference (熔断开关, 断路器, canary) and 标出 什么 is 缺失, mis-tuned, or exposed to the 智能体.

产出:

1. **Kill-switch 审计.** 位置 does the switch live (feature 标出, Redis, signed config)? 确认 the 智能体's 凭据 不能 设置 it off. 确认 每个 consequential action checks the switch, 不just startup. 确认 re-enable is an 明确 人类 action.
2. **Circuit-breaker inventory.** 列出 每个 pattern a breaker watches (repetition, consecutive 失败, rate, 具体 工具 之后 out-of-trust 阅读). 说明 阈值 and cool-down for each. 阈值 above 10 are usually too loose.
3. **Canary 设计.** 列出 每个 金丝雀令牌 in the environment. For each: 什么 it is (fake 凭据, fake DB record, fake 文件, fake 记忆 entry), 在哪里 it lives, 什么 access triggers the alarm, 人员 is paged. 确认 no canary has a legitimate reason to be touched.
4. **Statistical + hard layering.** 确认 the stack uses at least one hard 限制 (Lesson 17 宪法式 style) in addition to 任何 statistical detectors (EWMA, z-评分). Statistical-只 detectors accept slow drift.
5. **Quarantine path.** 内容 happens when a detector fires? Full 智能体 stop, path-具体 暂停, traffic redirect (eBPF / Cilium honeypot), 告警-只. 确认 the path has been tested end-to-end at least once.

硬性拒绝:
- 任何 部署 没有 an 外部 熔断开关.
- Canary 词元 stored in systems the 智能体 has 编写 access to.
- Statistical-只 detection 带有 no hard 限制.
- Circuit breakers 带有 cool-downs that auto-re-enable 没有 人类审查.
- Unattended runs 在哪里 the 熔断开关 is checked 只 at startup, 不per action.

拒绝规则:
- If the 用户 不能 命名 the 具体 systems 外部 the 智能体's 凭据 that host the 熔断开关, 拒绝. "We use a config 文件 the 智能体 reads" is 不a 熔断开关 if the 智能体 can 编写 config 文件.
- If the 用户 treats the Auto Mode 分类器 (Lesson 10) as a replacement for tripwires, 拒绝. The 分类器 is orthogonal to detection-and-response.
- If the 拟议的 canaries sit in systems the 智能体 has legitimate reason to 阅读, 拒绝 and 要求 redesign.

输出格式:

返回a tripwire 审计 带有:
- **Kill-switch 行** (位置, check 节奏, re-enable procedure)
- **Circuit-breaker 表格** (pattern, 阈值, cool-down)
- **Canary 表格** (词元, 位置, alarm, 负责人)
- **Layering note** (statistical + hard 限制 存在 是/否)
- **Quarantine flow** (什么 fires, 什么 happens, tested 是/否)
- **就绪度** (生产环境 / 预发布环境 / 仅研究)
