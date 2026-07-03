---
name: asp-enrichment
description: "创建 ASP enrichment。适用于把结构化分析、情报、资产上下文或调查结论保存到 case、alert 或 artifact。"
argument-hint: "create enrichment <target_id>"
compatibility: requires asp-cli >= 0.1.0
metadata:
  author: Funnywolf
  version: 1.0.0
  cli: asp
  category: cyber security
  tags: [enrichment, evidence, structured-data, investigation]
  documentation: https://asp.viperrtp.com/
---

# ASP Enrichment

当数据需要以结构化上下文形式保存回 ASP，并挂载到对应 case、alert 或 artifact 时，使用这个 skill。

默认使用用户的语言回复；如果用户没有明确语言偏好，使用中文。

## 前置检查

如果 ASP CLI 尚未配置或认证状态不明确，先运行 `/asp-setup`。也可以直接执行：

```bash
asp doctor --output json
```

## 适用场景

- 用户明确要求保存结构化分析、情报或调查结论。
- 用户想把 SIEM 发现、威胁情报、资产上下文或分析师判断附加到 case、alert 或 artifact。
- 用户已经形成稳定结论，需要后续流程、自动化或其他分析师复用。

## 运行规则

- Enrichment 是平台的结构化结果层，不是普通评论字段。
- 只有当用户明确要求持久化或“保存为结构化结果”时，才创建 enrichment。
- 人类可读交接、备注、解释性说明优先使用 `asp-comment`。
- `data` 必须是 JSON object，不要传普通文本或数组。
- payload 应紧凑、可操作，避免把整段调查叙事塞入 `data`。
- 查看对象本身时优先使用对象对应 skill；保存结果时才使用本 skill。

## 命令契约

### 创建简单 enrichment

```bash
asp enrichment create case_000001 --name "ti-summary" --type "Threat Intelligence" --value "8.8.8.8" --desc "该 IOC 在多个来源中显示高风险" --output json
```

### 创建带 JSON 数据的 enrichment

```bash
asp enrichment create artifact_000001 --name "ioc-context" --type "Threat Intelligence" --value "8.8.8.8" --uid "ti:8.8.8.8" --desc "威胁情报聚合结果" --data-json "{\"risk\":\"high\",\"sources\":[\"provider-a\"]}" --output json
```

### 从文件读取 JSON 数据

```bash
asp enrichment create alert_000001 --name "siem-scope" --type "Correlation" --data-file ./enrichment.json --output json
```

参数规则：

- `target_id` 必须以 `case_`、`alert_` 或 `artifact_` 开头。
- `name` 是简短 enrichment 标题。
- `type` 应使用 ASP enrichment 类型，例如 `Threat Intelligence`、`Reputation`、`Geo Location`、`DNS`、`Vulnerability`、`Asset`、`CMDB`、`Identity`、`Behavior`、`Detection`、`Correlation`、`History`、`Remediation`、`Observation`、`External Ticket`、`Other`。
- `value` 是主要被富化对象，例如 IOC、用户名、主机、case verdict 或 rule name。
- `uid` 是可选外部稳定标识，用于去重或追踪来源。
- `desc` 是简短可读摘要。
- `--data-json` 或 `--data-file` 必须表示 JSON object。

## 决策流程

1. 判断用户是否明确要求保存结构化结果。
2. 确认目标对象：`case_`、`alert_` 或 `artifact_`。
3. 将已有分析整理成紧凑字段：`name`、`type`、`value`、`uid`、`desc`、`data`。
4. 如果只是自然语言备注，转交 `asp-comment`。
5. 如果还没有稳定结论，不要创建 enrichment；先继续调查或说明缺口。
6. 执行 `asp enrichment create`。

## SOP

### 创建 enrichment

1. 确认 `target_id`。
2. 判断 enrichment 类型和主要 value。
3. 把用户的分析整理成紧凑 JSON object。
4. 调用 `asp enrichment create <target_id> ... --output json`。
5. 确认返回的 `enrichment_id`，并说明它已附加到哪个目标对象。

首选回复结构：

- `Target`：目标 ID。
- `Enrichment`：创建出的 enrichment ID。
- `Saved context`：保存了什么结构化信息。

## 澄清规则

- 缺少 `target_id` 时，询问。
- 用户只说“保存这个结果”但上下文中有多个可能目标时，询问保存到哪个 case、alert 或 artifact。
- enrichment 类型无法判断时，给出推荐类型并请求确认。
- `data` 无法组织成 JSON object 时，询问应保存哪些字段。

## 输出规则

- 保持简洁。
- 除非用户明确要求，否则不要输出原始 JSON。
- 明确说明保存了什么、附加到了哪里，以及它为什么有用。
- 不要把很长的原始调查文本塞进回复；只输出必要摘要和 ID。

## 失败处理

- 如果 ASP CLI 返回连接错误或超时，直接回复失败，提示用户运行 `asp doctor --output json`，检查 API URL、API key、网络连通性、用户状态和权限。不要尝试绕过认证。
- 如果目标对象不存在，直接说明。
- 如果 enrichment payload 不完整或 `data` 不是 JSON object，只问一个聚焦问题，不要猜测。
