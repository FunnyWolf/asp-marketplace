---
name: asp-comment-zh
description: '为 ASP case、alert、artifact、enrichment、knowledge 或 playbook run 添加评论，支持附件、回复和 @ 用户。'
argument-hint: 'comment <target_id> [body] [file_keys] | read file <file_key>'
compatibility: connect to asp mcp server
metadata:
  author: Funnywolf
  version: 0.2.0
  mcp-server: asp
  category: cyber security
  tags: [ comment, note, collaboration, investigation ]
  documentation: https://asp.viperrtp.com/
---

# ASP Comment

当用户要给 ASP 记录添加自然语言评论、分析师评论、交接说明、附件评论，或读取评论附件元数据时，使用这个 skill。

## 适用场景

- 用户要求添加 note、comment、分析观察、交接说明或调查备注。
- 内容是给人看的叙述文本，而不是结构化 enrichment 数据。
- 目标是已存在的 case、alert、artifact、enrichment、knowledge 或 playbook run。
- 用户要把已上传文件附加到评论，或根据 `file_key` 获取评论附件下载地址。

## 运行规则

- 叙述性备注、回复、@ 用户和附加已上传文件都使用 `add_comment`。
- 如果用户提供本地文件路径而不是 `file_key`，先通过 REST `POST /api/attachments/` 上传文件，拿返回的 `access_key` 作为 `file_key`。
- 如果用户要查看或下载某个文件，使用 `get_file(file_key)`；MCP 只返回文件元数据和下载地址，不返回文件内容。
- 如果用户要保存结构化数据、证据、风险评分、IOC 上下文或机器可读分析，改用 `asp-enrichment-zh`。
- 不要编造 target ID、comment ID 或 file_key。目标不明确时，询问精确记录 ID。

## MCP 工具契约

- `add_comment(target_id, body="", file_keys=None, parent_id=None, mentions=None)`
  - `target_id` 必须以下列前缀之一开头：`case_`、`alert_`、`artifact_`、`enrichment_`、`knowledge_`、`playbook_`。
  - `body` 可以为空；但空正文时必须提供至少一个 `file_key`。
  - `file_keys` 是附件 `access_key` / `file_key`，可传单个值、逗号分隔字符串、JSON 数组字符串或列表。
  - `parent_id` 用于回复评论，必须属于同一个目标资源。
  - `mentions` 优先使用用户名，兼容数字用户 ID。
  - comment 会以当前认证 MCP 用户身份创建。
- `get_file(file_key)`
  - `file_key` 是评论附件返回的 `file_key`，也就是附件 `access_key`。
  - 返回 `file_key`、`filename`、`size`、`content_type`、`download_url`。
  - 不返回文件正文、bytes、base64 或解析后的内容。

## SOP

1. 确认精确 `target_id`。
2. 提取 comment body、附件、回复目标和 @ 用户。尽量保留用户原意，只做必要格式整理。
3. 如果用户给的是本地文件路径，先调用 REST 附件上传接口；如果用户给的是 `file_key`，直接使用。
4. 调用 `add_comment(target_id=<target_id>, body=<body>, file_keys=<file_keys>, parent_id=<parent_id>, mentions=<mentions>)`。
5. 确认 comment 已添加，并给出 comment ID；如果附带附件，列出文件名和 `file_key`。

读取文件时：

1. 确认 `file_key`。
2. 调用 `get_file(file_key=<file_key>)`。
3. 返回文件名、类型、大小和下载地址。不要声称已经读取或解析文件内容。

首选回复结构：

- `Target`：目标 ID
- `Comment`：创建后的 comment ID
- `Attachments`：仅在有附件时列出文件名和 `file_key`

## 澄清规则

- 如果缺少或无法确定 `target_id`，询问。
- 如果既没有 comment body，也没有 `file_key` 或可上传文件，询问要写入的评论内容或要附加的文件。
- 如果用户说“这个文件”但没有给出本地路径或 `file_key`，询问文件来源。
- 如果用户说“保存这个结果”且内容是结构化证据，转向 enrichment 而不是 comment。

## 输出规则

- 保持简洁。
- 除非用户要求，不要重复很长的 comment body。

## 失败处理

- 如果 MCP 工具调用返回连接错误或超时，直接回复失败，提示用户检查 `ASP_MCP_URL`、`ASP_MCP_API_KEY`、ASGI `/api/mcp` 是否可访问，以及 API key 是否过期、用户是否被禁用。不要尝试重试或绕过。
- 如果目标记录不存在，直接说明。
- 如果 `file_key`、`parent_id` 或 `mentions` 无效，直接说明失败原因；不要静默跳过。
