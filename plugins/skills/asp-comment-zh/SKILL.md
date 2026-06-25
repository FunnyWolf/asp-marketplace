---
name: asp-comment-zh
description: '为 ASP case、alert、artifact、enrichment、knowledge 或 playbook run 添加自然语言评论。'
argument-hint: 'comment <target_id> <body>'
compatibility: connect to asp mcp server
metadata:
  author: Funnywolf
  version: 0.1.0
  mcp-server: asp
  category: cyber security
  tags: [ comment, note, collaboration, investigation ]
  documentation: https://asp.viperrtp.com/
---

# ASP Comment

当用户要给 ASP 记录添加自然语言评论、分析师评论或交接说明时，使用这个 skill。

## 适用场景

- 用户要求添加 note、comment、分析观察、交接说明或调查备注。
- 内容是给人看的叙述文本，而不是结构化 enrichment 数据。
- 目标是已存在的 case、alert、artifact、enrichment、knowledge 或 playbook run。

## 运行规则

- 叙述性备注使用 `add_comment`。
- 如果用户要保存结构化数据、证据、风险评分、IOC 上下文或机器可读分析，改用 `asp-enrichment-zh`。
- 不要编造 target ID。目标不明确时，询问精确记录 ID。

## MCP 工具契约

- `add_comment(target_id, body)`
  - `target_id` 必须以下列前缀之一开头：`case_`、`alert_`、`artifact_`、`enrichment_`、`knowledge_`、`playbook_`。
  - `body` 必填且不能为空。
  - comment 会以当前认证 MCP 用户身份创建。
  - 返回创建后的 comment `id`、`body`、`author`、`created_at`。

## SOP

1. 确认精确 `target_id`。
2. 提取 comment body。尽量保留用户原意，只做必要格式整理。
3. 调用 `add_comment(target_id=<target_id>, body=<body>)`。
4. 确认 comment 已添加，并给出 comment ID。

首选回复结构：

- `Target`：目标 ID
- `Comment`：创建后的 comment ID

## 澄清规则

- 如果缺少或无法确定 `target_id`，询问。
- 如果缺少 comment body，询问。
- 如果用户说“保存这个结果”且内容是结构化证据，转向 enrichment 而不是 comment。

## 输出规则

- 保持简洁。
- 除非用户要求，不要重复很长的 comment body。

## 失败处理

- 如果 MCP 工具调用返回连接错误或超时，直接回复失败，提示用户检查 `ASP_MCP_URL`、`ASP_MCP_API_KEY`、ASGI `/api/mcp` 是否可访问，以及 API key 是否过期、用户是否被禁用。不要尝试重试或绕过。
- 如果目标记录不存在，直接说明。
- 如果 body 为空，要求提供评论文本。
