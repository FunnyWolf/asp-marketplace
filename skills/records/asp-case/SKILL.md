---
name: asp-case
description: "管理 ASP 安全 case。适用于列出、查看、审查 case，或更新 case AI 分析字段。"
argument-hint: "list cases | show case <case_id> | update-ai <case_id>"
compatibility: requires asp-cli >= 0.1.0
metadata:
  author: Funnywolf
  version: 1.0.0
  cli: asp
  category: cyber security
  tags: [case-management, SOC, triage, investigation]
  documentation: https://asp.viperrtp.com/
---

# ASP Case

当用户要以 case 为中心开展 SOC 工作时，使用这个 skill。Case 是 ASP 中的核心调查对象，通常承载一个或多个 Alert，而 Alert 再关联一个或多个 Artifact。

默认使用用户的语言回复；如果用户没有明确语言偏好，使用中文。

## 前置检查

如果 ASP CLI 尚未配置或认证状态不明确，先运行 `/asp-setup`。也可以直接执行：

```bash
asp doctor --output json
```

## 适用场景

- 用户给出 case ID，希望查看、分诊、审查或总结 case。
- 用户希望按状态、严重级别、置信度、verdict、correlation UID、标题或标签查找 case。
- 用户想更新 case 的 AI 分析字段或 summary。
- 用户想围绕 case 添加评论或结构化 enrichment；此时本 skill 负责判断并转交 `asp-comment` 或 `asp-enrichment`。

## 运行规则

- 总结 case 数据时要服务于行动判断，而不是复述原始 JSON。
- 保持 case 作为主要视图。只有能帮助回答 case 问题时，才拉取关联 alert、comment、enrichment 或 playbook。
- `case_id` 是面向外部使用的记录标识，例如 `case_000001`。
- 不要编造 case、alert、artifact 之间的隐藏关系；只基于 CLI 返回的数据判断。
- 写操作只限 `asp case update-ai` 暴露的 AI 分析字段；不要声称可以修改分析师工作流字段。

## 命令契约

### 列出 case

```bash
asp case list --output json
```

支持的常用过滤参数：

```bash
asp case list --status "New" --severity "High" --confidence "Medium" --verdict "Suspicious" --correlation-uid "<uid>" --title "<keyword>" --tags "tag1,tag2" --page-size 20 --output json
```

- `status`：`New`、`In Progress`、`On Hold`、`Resolved`、`Closed`。
- `severity`：`Unknown`、`Informational`、`Low`、`Medium`、`High`、`Critical`。
- `confidence`：`Unknown`、`Low`、`Medium`、`High`。
- `verdict`：`Unknown`、`False Positive`、`True Positive`、`Disregard`、`Suspicious`、`Benign`、`Test`、`Insufficient Data`、`Security Risk`、`Managed Externally`、`Duplicate`、`Other`。
- 多个 tag 使用逗号分隔；多个过滤值也优先使用 CLI 支持的逗号分隔方式。
- 需要关联 alert 时加 `--include-related`。
- 需要评论时额外执行 `asp comment list <case_id> --output json`，不要假设 case 查询会自动带出 comments。

### 查看单个 case

```bash
asp case show case_000001 --include-related --output json
```

快速查看、不需要关联 alert 时：

```bash
asp case show case_000001 --no-include-related --output json
```

### 更新 AI 分析字段

仅当用户明确要求更新时执行：

```bash
asp case update-ai case_000001 --severity-ai High --confidence-ai Medium --impact-ai High --priority-ai High --verdict-ai Suspicious --summary "分析摘要" --output json
```

支持更新的字段：

- `severity_ai`
- `confidence_ai`
- `impact_ai`
- `priority_ai`
- `verdict_ai`
- `summary`

该命令不能更新分析师 `status`、分析师 `severity`、分析师 `verdict`、assignee 或工作流时间。

## 决策流程

1. 如果用户提供了具体 case ID，或要求 open/show/review/summarize 某个 case，调用 `asp case show <case_id> --include-related --output json`。
2. 如果用户只需要快速查看基本信息，调用 `asp case show <case_id> --no-include-related --output json`。
3. 如果用户要查找、浏览或对比 case，调用 `asp case list`，只使用 CLI 明确支持的过滤参数。
4. 如果用户要求查看评论，额外调用 `asp comment list <case_id> --output json`。
5. 如果用户要更新 AI 分析字段或 summary，使用 `asp case update-ai`，并且只携带用户明确要求修改的字段。
6. 如果用户要添加自然语言评论，转交 `asp-comment`。
7. 如果用户要保存结构化证据、评分或机器可读分析，转交 `asp-enrichment`。
8. 如果用户给出不受支持的过滤条件，忽略前先明确说明该过滤条件当前不支持。

## SOP

### 审查单个 case

1. 获取 case 详情和关联 alert。
2. 只在相关时获取 comments、playbook run 或其他上下文。
3. 如果找不到 case，直接说明。
4. 只展示与用户问题最相关的部分。
5. 当用户问“发生了什么”或“帮我理解这个 case”时，先给一段分析性总结，再给结构化细节。

首选回复结构：

- `Case`：case ID、标题、严重级别、状态、verdict、confidence、impact、priority、correlation UID、tags。
- `Timeline`：创建时间和关键更新时间。
- `Key Alerts`：只列最相关的告警，不默认列全。
- `Comments`：仅在检查过评论且相关时给出关键分析或交接点；附件只列文件名和 `file_key`。
- `Analyst / AI Notes`：在相关时给出 summary 和 AI 字段。

### 列出 case

1. 提取支持的过滤字段：`status`、`severity`、`confidence`、`verdict`、`correlation_uid`、`title`、`tags`、`page_size`。
2. 如果用户给出逗号分隔或自然语言列表，在调用 CLI 前规范化。
3. 调用 `asp case list --output json` 并附加必要过滤条件。
4. 解析返回的 JSON。
5. 用紧凑对比视图呈现。
6. 如果结果很多，建议下一步最有价值的过滤条件，而不是倾倒全部结果。

首选列表结构：

| Case ID | Title | Severity | Status | Verdict | Confidence | Priority | Created |
| --- | --- | --- | --- | --- | --- | --- | --- |

### 更新 case AI 分析

1. 确认 `case_id`。
2. 只提取用户明确要求修改的字段。
3. 在调用 CLI 前校验枚举值是否能映射到 ASP 支持值。
4. 仅携带变更字段调用 `asp case update-ai`。
5. 用简短变更记录风格确认更新结果。
6. 如果用户可能还要核实结果，建议重新获取 case。

首选回复结构：

- `Updated case`：case ID。
- `Changed fields`：只列实际提交的字段。

## 澄清规则

- 缺少 `case_id` 且任务需要单个 case 时，询问。
- 请求值无法清晰映射到 ASP 枚举时，询问。
- 用户要求修改 close、resolve、分析师 verdict、分析师 severity、assignee 等字段时，说明当前 CLI 不暴露这些字段；可改为保存 AI 建议或添加 comment。
- 用户给出广义审查请求，例如“show recent important cases”，先从 `asp case list` 开始，不强迫用户先选操作。

## 输出规则

- 保持简洁，优先使用面向分析师的措辞，而不是 schema 措辞。
- 除非用户明确要求，否则不要输出原始 JSON。
- 表格保持精简；匹配很多时展示最有价值的子集并说明总量或翻页方式。
- 如果一次审查用了多个 CLI 调用，把结果合并成一个连贯的 case 叙事，不要逐个调用回显。
- 明确指出阻塞项：case 不存在、不支持的过滤条件、无效枚举值、认证失败或权限不足。

## 失败处理

- 如果 ASP CLI 返回连接错误或超时，直接回复失败，提示用户运行 `asp doctor --output json`，检查 API URL、API key、网络连通性、用户状态和权限。不要尝试绕过认证。
- 如果 case 不存在，直接说明。
- 如果过滤无结果，直接说明并建议最可能有用的收敛方式。
- 如果更新目标不明确，只问一个聚焦问题，不要猜测。
