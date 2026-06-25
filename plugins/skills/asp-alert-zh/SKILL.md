---
name: asp-alert-zh
description: '查看 ASP 告警并进行分诊分析。'
argument-hint: 'review alert <alert_id> | list alerts [filters]'
compatibility: connect to asp mcp server
metadata:
  author: Funnywolf
  version: 0.1.0
  mcp-server: asp
  category: cyber security
  tags: [ alert-management, soc, triage, investigation ]
  documentation: https://asp.viperrtp.com/
---

# ASP Alert

当用户要围绕 ASP 告警开展 SOC 分析工作时，使用这个 skill。
Alert 是 ASP 中的二级数据,每个 Alert 都会挂载到一个 Case,一个 Alert 会挂载一个或多个 Artifact。

## 适用场景

- 用户给出一个告警 ID，希望快速查看、审查或总结。
- 用户希望按状态、严重级别、置信度或 correlation UID 查找告警。
- 用户想在分析后把 enrichment 附加到告警。

## 运行规则

- 回复要聚焦于分诊价值，而不是原样回显 schema 字段。
- Alert 当前为只读接口，如需更新分析结果请使用 enrichment。
- 如果用户需要保存分析结果或结构化上下文到告警上，使用 `asp-enrichment-zh` skill。
- 如果用户需要在告警上添加自然语言评论，使用 `asp-comment-zh` skill。

## 补充信息

- `alert_id` 是 MCP 面向外部使用的告警记录标识，例如 `alert_000001`。

## 决策流程

1. 如果用户提供了具体告警 ID，或要求 open/show/review/summarize 某条告警，调用 `list_alerts(alert_id=<id>, limit=1, include_related=True)`。
2. 如果用户要浏览或对比多条告警，使用 `list_alerts(..., include_related=False)` 和支持的过滤条件。
3. 如果用户要附加分析结果、情报或结构化上下文，使用 `asp-enrichment-zh` skill。

## MCP 工具契约

- `list_alerts(alert_id=None, status=None, severity=None, confidence=None, correlation_uid=None, include_related=False, limit=10)`
  - `alert_id` 是 `alert_000001` 这类可读 ID。
  - `status`：`Unknown`、`New`、`In Progress`、`Suppressed`、`Resolved`、`Archived`、`Deleted`、`Other`。
  - `severity`：`Unknown`、`Informational`、`Low`、`Medium`、`High`、`Critical`。
  - `confidence`：`Unknown`、`Low`、`Medium`、`High`。
  - `correlation_uid` 用于匹配同一 case correlation 下的告警。
  - `limit` 会被限制在 1-100。
  - `include_related=False` 返回紧凑告警记录。只有审查具体某一条且需要 artifacts、enrichments 时才设为 `True`。

## SOP

### 审查单条告警

1. 如果用户要求审查、分析或查看告警详情，调用 `list_alerts(alert_id=<id>, limit=1, include_related=True)`。
2. 如果只需要快速查看告警基本信息，调用 `list_alerts(alert_id=<id>, limit=1, include_related=False)` 即可。
3. 如果结果为空，直接说明找不到该告警。
4. 解析第一条 JSON 记录。
5. 只呈现最有价值的分诊字段。

首选回复结构：

- `Alert`：alert ID、标题或名称、严重级别、状态、置信度、correlation UID。
- `Timeline`：创建时间。
- `Key Context`：source UID、rule ID、rule name、correlation UID、关联 case ID、artifacts、enrichments 中相关的部分。
- `Assessment`：简短分诊判断。

### 列出告警

1. 提取支持的过滤字段：`alert_id`、`status`、`severity`、`confidence`、`correlation_uid`、`limit`。
2. 在调用 MCP 前，把自然语言过滤条件规范化。
3. 调用 `list_alerts`。
4. 解析返回的 JSON 字符串。
5. 以紧凑对比视图呈现。

首选回复结构：

| Alert ID | Title | Severity | Status | Confidence | Created | Rule Name |
|----------|-------|----------|--------|------------|---------|-----------|

然后在需要时补一句简短解释。

## 澄清规则

- 只有在缺少告警相关操作所需参数时才询问 `alert_id`。
- 只有当请求值不能清晰映射到 ASP 枚举时，才要求用户澄清枚举值。

## 输出规则

- 保持简洁。
- 除非用户明确要求，否则不要输出原始 JSON。
- 优先使用分诊语义，而不是 schema 语义。
- 清楚指出阻塞项：告警不存在、不支持的过滤条件、无效枚举值。

## 失败处理

- 如果 MCP 工具调用返回连接错误或超时，直接回复失败，提示用户检查 `ASP_MCP_URL`、`ASP_MCP_API_KEY`、ASGI `/api/mcp` 是否可访问，以及 API key 是否过期、用户是否被禁用。不要尝试重试或绕过。
- 如果告警不存在，直接说明。
- 如果过滤无结果，直接说明并建议最有用的收敛方式。
