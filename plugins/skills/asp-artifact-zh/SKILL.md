---
name: asp-artifact-zh
description: '按 IOC 查找 artifact'
argument-hint: 'review artifact <artifact_id> | list artifacts [filters]'
compatibility: connect to asp mcp server
metadata:
  author: Funnywolf
  version: 0.1.0
  mcp-server: asp
  category: cyber security
  tags: [ artifact, pivot, enrichment, investigation ]
  documentation: https://asp.viperrtp.com/
---

# ASP Artifact

当用户要围绕 artifact 进行调查分析时，使用这个 skill。
artifact 由系统自动创建，用户只能查询和分析已有 artifact，并通过 enrichment 保存分析结果。

## 适用场景

- 用户想按 value、type、role 或 artifact ID 查找 artifact。

## 运行规则

- 把 artifact 视为平台里的最小调查对象。
- 查询和审查时使用 `list_artifacts`。
- 如需保存分析结果到 artifact，使用 `asp-enrichment-zh` skill。

## 补充信息

- `artifact_id` 是 MCP 面向外部使用的 artifact 记录标识，例如 `artifact_000001`。

## 决策流程

1. 如果用户要审查具体某个 artifact，调用 `list_artifacts(artifact_id=<id>, limit=1, include_related=True)`。
2. 如果用户要浏览或对比 artifact，调用 `list_artifacts(..., include_related=False)`。
3. 如果用户要为 artifact 附加情报、分析笔记或结构化分析，使用 `asp-enrichment-zh` skill。
4. 如果用户正从 artifact 出发进行调查，把 artifact 作为最小调查对象，只在必要时建议下一个最有价值的跳转点。

## MCP 工具契约

- `list_artifacts(artifact_id=None, type=None, role=None, value=None, include_related=False, limit=10)`
  - `artifact_id` 是 `artifact_000001` 这类可读 ID。
  - `type` 可使用 ASP artifact 类型值，例如 `IP Address`、`Hostname`、`User Name`、`Email Address`、`URL String`、`Hash`、`File Name`、`Process Name`、`Resource UID`、`Port`、`Subnet`、`Command Line`、`CVE`、`Account`、`Resource`、`File Path`、`Device`、`Registry Path`、`Other` 等平台值。
  - `role`：`Unknown`、`Target`、`Actor`、`Affected`、`Related`、`Other`。
  - `value` 是 artifact 值的精确匹配。
  - `type` 和 `role` 可传字符串、逗号分隔字符串、JSON 数组字符串或数组。
  - `include_related=False` 返回紧凑 artifact 记录。只有审查具体某一条且需要 enrichments 时才设为 `True`。
  - `limit` 会被限制在 1-100。
  - 当前 MCP 工具没有 owner 过滤条件。

## SOP

### 列出 Artifact

1. 从请求中提取最窄且最有用的过滤条件。
2. 广泛查询/列表使用 `include_related=False`；只有具体某条 artifact 需要 enrichment 上下文时才用 `include_related=True`。
3. 调用 `list_artifacts`。
4. 解析返回的 JSON 字符串。
5. 以紧凑的 artifact 视图呈现；如果用户大概率下一步要附加或复用该 artifact，则显式展示 `artifact_id`。

首选回复结构：

| Artifact ID | Value | Type | Role | Summary |
|-------------|-------|------|------|---------|

然后在需要时补一句简短解释。

## 澄清规则

- 只有当用户要 enrich 现有 artifact 却未提供时，才询问 `artifact_id`。

## 输出规则

- 保持简洁。
- 除非用户明确要求，否则不要输出原始 JSON。
- 优先使用 pivot 语义，而不是存储语义。
- 当匹配的 artifact 很多时，展示最有价值的一小部分，并简要说明整体模式。

## 失败处理

- 如果 MCP 工具调用返回连接错误或超时，直接回复失败，提示用户检查 `ASP_MCP_URL`、`ASP_MCP_API_KEY`、ASGI `/api/mcp` 是否可访问，以及 API key 是否过期、用户是否被禁用。不要尝试重试或绕过。
- 如果没有匹配的 artifact，直接说明，并建议最有用的收敛方式。
- 如果目标 artifact 不存在，直接说明。
