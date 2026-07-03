---
name: asp-threat-hunting
description: "编排 ASP 威胁狩猎。适用于围绕假设、IOC、TTP 或可疑活动进行有边界的 SIEM hunt。"
argument-hint: "hunt for <hypothesis or IOC>"
compatibility: requires asp-cli >= 0.1.0
disable-model-invocation: true
metadata:
  author: Funnywolf
  version: 1.0.0
  cli: asp
  category: cyber security
  tags: [threat-hunting, SIEM, IOC, investigation]
  documentation: https://asp.viperrtp.com/
---

# ASP Threat Hunting

当用户要主动搜索威胁痕迹，而不是只做单个对象查询或响应单条告警时，使用这个用户显式调用的编排 skill。威胁狩猎必须有明确假设、范围和时间窗口。

默认使用用户的语言回复；如果用户没有明确语言偏好，使用中文。

## 前置检查

如果 ASP CLI 尚未配置或认证状态不明确，先运行 `/asp-setup`。也可以直接执行：

```bash
asp doctor --output json
```

## 何时使用

- 用户要围绕一个威胁假设主动搜索证据。
- 用户给出 IOC、TTP、攻击行为或情报线索，希望验证环境中是否出现过。
- 用户要做范围界定、时间线还原、横向扩展或跨源验证。
- 用户要输出一份有证据支撑的威胁狩猎报告。

## 不要使用

- 用户只是查单个 IOC 声誉，应使用 `asp-threat-intelligence`。
- 用户只是查看一个 artifact，应使用 `asp-artifact`。
- 用户只是调查一个 case，应使用 `asp-case-investigation`。
- 用户没有提供假设、IOC、行为或范围，且无法从上下文推导最小可行 hunt。

## 可用能力

- `asp siem schema list` / `asp siem schema show` / `asp siem fields discover`：了解有哪些数据源、字段叫什么、样例值是什么。
- `asp siem search keyword` / `asp siem query adaptive`：搜索日志和验证假设。
- `asp siem query spl` / `asp siem query esql`：执行复杂原始查询。
- `asp ti query`：查询 IOC 声誉，帮助形成或扩展假设。

默认不使用 case、alert、knowledge、comment、enrichment 或 playbook。只有用户明确要求保存结果或关联到事件响应流程时，才转交对应 skill。

## 狩猎原则

- 每个结论必须有工具输出支撑，不编造证据。
- 不猜字段名；字段不确定时先探索 schema 或发现字段。
- SIEM 查询必须有明确时间范围，且时间必须带时区。
- 不做无边界 hunt。
- 每轮只推进 1-3 个具体问题，最多 5 轮。
- 优先小步迭代查询，而不是一次倾倒大范围数据。
- 无发现也是发现，但必须说明查询范围和限制。
- 默认不保存 comment 或 enrichment；只有用户明确要求时才持久化。

## 命令入口

### 探索数据源

```bash
asp siem schema list --output json
asp siem schema show <index_name> --output json
```

### 发现字段

```bash
asp siem fields discover <index_name> <backend> --from <start> --to <end> --output json
```

### 关键词搜索

```bash
asp siem search keyword "<keyword>" --from <start> --to <end> --output json
```

### 自适应查询

```bash
asp siem query adaptive <index_name> --from <start> --to <end> --filters-json "<json>" --output json
```

### 原始查询

```bash
asp siem query spl "<spl_query>" --from <start> --to <end> --output json
asp siem query esql "<esql_query>" --from <start> --to <end> --output json
```

### IOC 富化

```bash
asp ti query <indicator> --artifact-type <type> --output json
```

## 假设构建

用 ABLE 方法构建 hunt 假设：

- **Actor**：谁。
- **Behavior**：做什么。
- **Location**：在哪。
- **Evidence**：应该看到什么证据。

假设格式：

```text
如果 <Actor> 使用了 <ATT&CK 技术或行为> 在 <Location> 执行 <Behavior>，我们应该在 <数据源> 中看到 <Evidence>。
```

常见假设示例：

| 攻击意图 | 假设 | 需要的数据源 | 关键证据 |
| --- | --- | --- | --- |
| 进程注入 | 如果攻击者使用 T1055，端点日志中应出现非常规进程对敏感进程的注入行为 | Sysmon / EDR | 目标进程、调用者进程、线程起始地址 |
| 横向移动 | 如果攻击者使用 T1021，网络日志中应出现 SMB/RDP/WinRM 访问和远程服务或登录事件 | 网络流 + 系统事件 | 登录事件、SMB 连接对、服务名 |
| 持久化 | 如果攻击者使用 T1053，系统日志中应出现计划任务创建或修改 | Sysmon / 安全日志 | 任务名、触发器、执行命令 |
| C2 信标 | 如果存在 C2 信标，网络日志中应出现定期外连固定 IP 或域名的模式 | 网络流 + DNS | 连接间隔、目标 IP/域名、数据量 |

情报驱动时，先用 `asp ti query` 理解 IOC 是什么，再基于情报结果选择对应假设方向。

## 推荐流程

### 1. 明确 hunt 范围

确认：

- hunt 假设或 IOC。
- 目标环境、数据源或索引。
- 时间范围。
- 预期证据。
- 成功或停止条件。

用户未指定时间时，不要静默做无界查询。可建议：

- 普通 hunt：最近 7 天。
- 情报驱动回溯：最近 30 天。
- 用户给出“昨天”等相对时间：根据用户时区换算；时区不明确时先询问。

### 2. 了解环境

不要猜字段名。

1. 使用 `asp siem schema list --output json` 查看数据源。
2. 使用 `asp siem schema show <index_name> --output json` 或 `asp siem fields discover` 查看字段。
3. 根据实际字段调整查询计划。

### 3. 搜索验证

把假设变成查询，执行并观察结果。

工具选择：

- 线索是关键词或 IOC：使用 `asp siem search keyword`。
- 已知 index 和字段：使用 `asp siem query adaptive`。
- 需要复杂逻辑、管道或高级聚合：使用 `asp siem query spl` 或 `asp siem query esql`。
- 搜到新 IOC：使用 `asp ti query` 富化，然后决定是否继续追查。

提前终止条件：

- 假设已被明确回答。
- 连续两轮无新发现。
- 所有可行方向穷尽。
- 缺少关键字段或数据源，无法继续验证。

### 4. 横向扩展

从发现的 IOC 或实体出发，沿三个方向扩展：

| 扩展方向 | 动作 | 示例 |
| --- | --- | --- |
| 同源关联 | 搜索同一 IOC 在不同时间段、不同目标上的出现 | 恶意 IP 的历史连接 |
| 同主机关联 | 搜索同一主机上的其他可疑活动 | 攻击时间段内的进程创建和网络连接 |
| 同用户关联 | 搜索同一用户的其他操作 | 登录、命令执行、文件访问 |
| 跨源交叉 | 端点日志与网络日志交叉验证 | 进程创建事件与出站连接关联 |

对发现的新 IOC，可先用 `asp ti query` 富化，再决定是否继续扩展。

### 5. 收敛

当假设被回答、连续两轮无新发现，或所有方向穷尽时，停止搜索。

优先级判断参考“痛苦金字塔”：

- IOC（IP、hash）容易被替换，价值较低。
- TTP（攻击手法、行为模式）更难替换，价值较高。
- 狩猎目标应尽量提升到 TTP 层面，而不是停留在 IOC 匹配。

## 输出报告

狩猎结束后输出 Markdown 报告：

```markdown
# 威胁狩猎报告

## 结论

[确认威胁 / 可疑活动 / 未发现]  置信度：[高/中/低]

一段话总结。

## 假设

本次狩猎验证了什么。

## 范围

时间范围、数据源、索引和关键查询条件。

## 过程

怎么做的，每次转向为什么。

## 发现

每个发现：发现了什么、证据是什么、涉及哪些 IOC 或实体。

## ATT&CK 映射

发现 → 战术 → 技术 → 日志中可观测到的具体字段和值。

## 可见性缺口

哪些攻击步骤现有日志覆盖不到。

## 建议

- 是否需要升级为安全事件。
- 是否需要隔离主机、阻断 IP、重置凭证或执行其他处置。
- 检测规则调整建议。
- 日志采集改进建议。
```

证据中的关键原始值（时间戳、主机名、进程名、命令行、父进程、用户、IOC）用代码块输出，不要用表格硬塞长字段。

## 澄清规则

- 缺少 hunt 假设、IOC 或行为线索时，询问。
- 缺少时间范围时，询问或建议最小默认范围并等待确认。
- 时区不明确时，询问。
- 数据源或字段不明确时，先 schema 探索，不要求用户直接给字段名。
- 用户要求“全面查一下”时，先要求限定范围、时间和目标，而不是无界搜索。

## 输出规则

- 保持聚焦，不试图验证所有攻击技术。
- 每个结论必须有查询结果支撑。
- 默认不要倾倒所有日志。
- 无发现时也要说明查了什么、没发现什么、限制是什么。
- 不保存 comment 或 enrichment，除非用户明确要求。

## 失败处理

- 如果 ASP CLI 返回连接错误或超时，直接回复失败，提示用户运行 `asp doctor --output json`，检查 API URL、API key、网络连通性、用户状态和权限。不要尝试绕过认证。
- 如果字段不存在或数据源不可用，说明可见性缺口。
- 如果查询结果为空，说明查询范围，并建议扩大时间或放宽条件。
- 如果查询结果过多，先收窄时间、index 或关键词，再继续。
