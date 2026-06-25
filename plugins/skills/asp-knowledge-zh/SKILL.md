---
name: asp-knowledge-zh
description: 'ASP 平台内部 Knowledge 记录的检索与维护。支持按关键词搜索知识条目，或按 ID 更新记录。'
argument-hint: 'search knowledge <query> | update knowledge <knowledge_id> <fields>'
compatibility: connect to asp mcp server
metadata:
  author: Funnywolf
  version: 0.1.0
  mcp-server: asp
  category: cyber security
  tags: [ knowledge, memory, investigation ]
  documentation: https://asp.viperrtp.com/
---

# ASP Knowledge

当用户要在 ASP 中检索或维护内部知识库时，使用这个 skill。

## 设计思路

Knowledge 是数据库中的知识记录，核心字段包括 `title`、`body`、`tags`、`source`、`expires_at`。

- `title` — 知识标题
- `body` — 知识主体内容，支持 Markdown
- `tags` — 标签，用于筛选和分类
- `source` — 知识来源，`Manual` 为手动创建，`Case` 为从结案 Case 中自动提取
- `expires_at` — 过期时间，留空表示永久有效

`search_knowledge` 会匹配 title、body、tags。当前 MCP 工具按数据库中匹配结果返回，不要声称会自动过滤过期记录。

## 适用场景

- 用户想通过关键词在知识库中查找相关内容。
- 用户想更新知识条目的标题、内容、标签或过期时间。

## 运行规则

- 如果用户是在找相关知识内容，尤其是给出主题、问题描述、症状或自然语言查询，使用 `search_knowledge`。
- 如果用户要修改已知知识条目的内容，使用 `update_knowledge`。

## 决策流程

1. 如果用户要搜索知识内容，使用 `search_knowledge`。
2. 如果用户要修改已知记录，调用 `update_knowledge`。

## MCP 工具契约

- `search_knowledge(keyword, limit=10)`
  - `keyword` 可传字符串、逗号分隔字符串、JSON 数组字符串或数组。多个关键词会在 title、body、tags 上做 OR 匹配。
  - `limit` 会被限制在 1-100。
- `update_knowledge(knowledge_id, title=None, body=None, expires_at=None, tags=None)`
  - `knowledge_id` 是 `knowledge_000001` 这类可读 ID。
  - `body` 支持 Markdown。用户提供结构化内容时，应保留标题、列表、表格、代码块、链接等 Markdown 格式。
  - `expires_at` 必须是带时区的 ISO 8601，例如 `2026-06-23T12:00:00Z` 或 `2026-06-23T20:00:00+08:00`。
  - `tags` 可传字符串、逗号分隔字符串、JSON 数组字符串或数组。

## SOP

### 搜索 Knowledge

1. 将用户问题、关键词或场景描述整理为搜索关键词。
2. 调用 `search_knowledge`。
3. 返回最相关的少量结果，优先给出直接相关的知识标题和简短说明。
4. 除非用户明确要求，否则不要展开完整 body。

首选回复结构：

| Knowledge ID | Title | Tags |
|--------------|-------|------|

然后补一句这些结果与查询的关系。

### 更新 Knowledge

1. 要求提供 `knowledge_id`。
2. 只提取用户明确要求修改的字段：`title`、`body`、`expires_at`、`tags`。将 `body` 视为支持 Markdown 的内容。
3. 仅带变更字段调用 `update_knowledge`。
4. 如果结果为 `None`，说明找不到该知识记录。
5. 只确认实际变更的字段。

首选回复结构：

- `Updated knowledge`：knowledge ID

## 澄清规则

- 只有在用户要更新特定记录但未提供时，才询问 `knowledge_id`。
- 只有当用户的意图不能清晰映射到 `search_knowledge` 或 `update_knowledge` 时，才补问一个聚焦问题。

## 输出规则

- 保持简洁。
- 除非用户明确要求，否则不要输出完整 knowledge body。
- 当匹配记录很多时，展示最有价值的子集，并简要说明整体模式。

## 失败处理

- 如果 MCP 工具调用返回连接错误或超时，直接回复失败，提示用户检查 `ASP_MCP_URL`、`ASP_MCP_API_KEY`、ASGI `/api/mcp` 是否可访问，以及 API key 是否过期、用户是否被禁用。不要尝试重试或绕过。
- 如果 `search_knowledge` 没有结果，直接说明，并提示用户换一组关键词。
- 如果要更新的记录不存在，直接说明。
