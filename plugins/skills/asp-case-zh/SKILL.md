---
name: asp-case-zh
description: '管理 ASP 安全 case。适用于审查 case、列出 case、查看关联数据，或更新 case AI 分析字段、summary、备注和 enrichment。'
argument-hint: 'review case <case_id> | list cases [filters] | update case <case_id> <fields>'
compatibility: connect to asp mcp server
metadata:
  author: Funnywolf
  version: 0.3.1
  mcp-server: asp
  category: cyber security
  tags: [ case-management, soc, triage, investigation ]
  documentation: https://asp.viperrtp.com/
---

# ASP Case

当用户要以 case 为中心开展 SOC 工作时，使用这个 skill。
Case 是 ASP 中的核心调查对象，一个 Case 会挂载一个或多个 Alert, 一个 Alert 挂载一个或多个 Artifact,所以用户通常调查过程只操作
Case。

## 适用场景

- 用户给出一个 case ID，希望查看、分诊或快速总结。
- 用户希望按状态、严重级别、置信度、verdict、correlation UID、标题或标签查找 case。
- 用户想更新 case AI 分析字段或 summary。
- 用户想把 enrichment 或结构化分析附加到 case。

## 运行规则

- 总结 case 数据时要服务于行动判断，而不是输出原始 schema。
- 保持 case 作为用户的主要视图。只有在能帮助回答 case 问题时，才拉取相关告警、评论、富化和 playbook；评论必须显式使用 `include_comments=True`。

## 补充信息

- `case_id` 是 MCP 面向外部使用的 case 记录标识，例如 `case_000001`。

## 决策流程

1. 如果用户提供了具体 case ID，或说要 open/show/review/summarize 某个 case，调用 `list_cases(case_id=<id>, limit=1, include_related=True, include_comments=True)`。
2. 如果用户要查找、浏览或对比 case，默认使用 `list_cases(..., include_related=False, include_comments=False)`；只有需要 alerts、enrichments、playbooks 时才设 `include_related=True`，需要 comments 时才设 `include_comments=True`。
3. 如果用户要更新 AI 分析字段或 summary，使用 `update_case`。
4. 如果用户要添加自然语言评论，使用 `asp-comment-zh` skill。
5. 如果用户给出了多个过滤条件，只应用 ASP 直接支持的部分，并明确说明不支持的过滤条件。
6. 如果用户要把 enrichment 或结构化分析附加到 case，使用 `asp-enrichment-zh` skill。

## MCP 工具契约

- `list_cases(case_id=None, status=None, severity=None, confidence=None, verdict=None, correlation_uid=None, title=None, tags=None, include_related=True, limit=10, include_comments=False, comments_limit=20)`
  - `case_id` 是 `case_000001` 这类可读 ID。
  - `status`：`New`、`In Progress`、`On Hold`、`Resolved`、`Closed`。
  - `severity`：`Unknown`、`Informational`、`Low`、`Medium`、`High`、`Critical`。
  - `confidence`：`Unknown`、`Low`、`Medium`、`High`。
  - `verdict`：`Unknown`、`False Positive`、`True Positive`、`Disregard`、`Suspicious`、`Benign`、`Test`、`Insufficient Data`、`Security Risk`、`Managed Externally`、`Duplicate`、`Other`。
  - `tags` 可传字符串、逗号分隔字符串、JSON 数组字符串或数组；多个 tag 会全部参与过滤。
  - `include_related=True` 会包含 alerts、enrichments、playbooks；不会隐式包含 comments。
  - `include_comments=True` 会包含最近的 comments；`comments_limit` 默认 20，最大 50。
  - comment 附件只返回 `file_key`、文件名、大小、类型和下载地址；需要下载文件时使用 `asp-file-zh` / `get_file`。
  - `limit` 会被限制在 1-100。
- `update_case(case_id, severity_ai=None, confidence_ai=None, impact_ai=None, priority_ai=None, verdict_ai=None, summary=None)`
  - 该工具只更新 AI 评估字段和 `summary`；不能更新分析师 `status`、分析师 `severity`、分析师 `verdict`、assignee 或工作流时间。
  - AI severity/impact/priority 可用 `Unknown`、`Low`、`Medium`、`High`、`Critical`；AI confidence 可用 `Unknown`、`Low`、`Medium`、`High`；AI verdict 使用上面的 verdict 值。

## SOP

### 审查单个 Case

1. 如果用户要求审查、分析或查看 case 详情，调用 `list_cases(case_id=<id>, limit=1, include_related=True, include_comments=True)` 获取关联数据（alerts、enrichments、playbooks）和最近 comments。
2. 如果只需要快速查看 case 基本信息，调用 `list_cases(case_id=<id>, limit=1, include_related=False, include_comments=False)` 即可。
3. 如果结果为空，直接说明找不到该 case。
4. 只展示与用户请求最相关的部分。
5. 只有在确实影响用户目标时，才强调缺失字段或可疑字段。

首选回复结构：

- `Case`：case ID、标题、严重级别、状态、verdict、confidence、impact、priority、correlation UID、tags。
- `Timeline`：创建时间。
- `Key Alerts`：只列最相关的告警，不默认列全。
- `Comments`：仅在相关时给出关键分析或系统评论点；附件只列文件名和 `file_key`。
- `Analyst / AI Notes`：在相关时给出 comment、summary 和 AI 字段。

当用户问“发生了什么”或“帮我理解这个 case”时，先给一段简短分析性总结，再给结构化细节。

### 列出 Case

1. 提取支持的过滤字段：`case_id`、`status`、`severity`、`confidence`、`verdict`、`correlation_uid`、`title`、`tags`、`limit`。
2. 如果用户给出逗号分隔或自然语言列表，在调用 MCP 前先规范化。
3. 调用 `list_cases`。
4. 解析返回的 JSON 字符串。
5. 以紧凑对比视图呈现。
6. 如果结果很多，建议下一步最有价值的过滤条件，而不是直接倾倒大量结果。

首选回复结构：

| Case ID | Title | Severity | Status | Verdict | Confidence | Priority | Created |
|---------|-------|----------|--------|---------|------------|----------|---------|

然后在需要时补一句简短解释，例如：

- “Most matching cases are still in progress.”
- “High-severity cases share a tag or correlation UID.”
- “No matching cases were found.”

### 更新 Case

1. 要求提供 `case_id`。
2. 只提取用户明确要求修改的字段。
3. 在调用 MCP 前校验请求里的枚举值。
4. 仅携带变更字段调用 `update_case`。
5. 如果结果为 `None`，说明找不到该 case。
6. 用简短变更记录风格确认更新结果。
7. 如果用户可能还要核实结果，建议重新获取 case。

支持更新的字段：

- `severity_ai`
- `confidence_ai`
- `impact_ai`
- `priority_ai`
- `verdict_ai`
- `summary`

自然语言评论请使用 `asp-comment-zh`。`status`、分析师 `severity`、分析师 `verdict` 等工作流字段当前没有 MCP 更新工具。

首选回复结构：

- `Updated case`：case ID
- `Changed fields`：只列实际提交的字段

## 澄清规则

- 只有在缺失时才询问 `case_id`。
- 只有当请求值无法清晰映射到 ASP 枚举时，才要求用户澄清。
- 如果用户要求修改 close/resolve/分析师 verdict 等工作流字段，说明当前 MCP 更新工具不暴露这些字段；可改为保存 AI 建议或添加 comment。
- 如果用户给出的是广义审查请求，比如“show recent important cases”，先从 `list_cases` 开始，不要强迫用户先选操作。

## 输出规则

- 保持简洁。
- 除非用户明确要求，否则不要输出原始 JSON。
- 优先使用面向分析师的措辞，而不是 schema 措辞。
- 表格保持精简；当匹配很多时，展示最有价值的子集并说明总量。
- 如果一次审查用了多个 MCP 调用，要把结果合并成一个连贯的 case 叙事，而不是逐个调用回显。
- 明确指出阻塞项：case 不存在、不支持的过滤条件、无效枚举值。

## 失败处理

- 如果 MCP 工具调用返回连接错误或超时，直接回复失败，提示用户检查 `ASP_MCP_URL`、`ASP_MCP_API_KEY`、ASGI `/api/mcp` 是否可访问，以及 API key 是否过期、用户是否被禁用。不要尝试重试或绕过。
- 如果 case 不存在，直接说明。
- 如果过滤无结果，直接说明并建议最可能有用的收敛方式。
- 如果更新目标不明确，只问一个聚焦问题，不要猜测。
