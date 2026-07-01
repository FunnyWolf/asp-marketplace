---
name: asp-file-zh
description: '通过 ASP MCP file_key 获取文件元数据和下载地址，并说明附件上传流程。'
argument-hint: 'get file <file_key> | upload file <path>'
compatibility: connect to asp mcp server
metadata:
  author: Funnywolf
  version: 0.1.0
  mcp-server: asp
  category: cyber security
  tags: [ file, attachment, download, mcp ]
  documentation: https://asp.viperrtp.com/
---

# ASP File

当用户要根据 `file_key` 获取 ASP 文件下载地址，或需要把文件上传后用于评论附件时，使用这个 skill。

## 适用场景

- 用户从评论附件或其他 ASP 响应中拿到了 `file_key`，希望下载或查看文件元数据。
- 用户要把本地文件上传到 ASP，并把返回的文件键用于后续评论。
- 用户询问 MCP 是否能读取评论附件文件。

## 运行规则

- MCP 的统一文件入口是 `get_file(file_key)`。
- `get_file` 只返回元数据和下载地址，不返回文件内容、文本、bytes 或 base64。
- 如果需要解析文件内容，明确说明需要由用户代码或客户端通过 `download_url` 下载后自行处理。
- 上传文件不通过 MCP 工具完成；使用 REST `POST /api/attachments/`，字段名为 `file`，认证仍使用 `Authorization: Api-Key <key>`。
- 上传返回的 `access_key` 就是后续 MCP 工具使用的 `file_key`。
- 不要编造 `file_key` 或下载地址。

## MCP / REST 契约

- `get_file(file_key)`
  - 返回 `file_key`、`filename`、`size`、`content_type`、`download_url`。
  - 找不到文件时会失败，不要使用猜测值重试。
- `POST /api/attachments/`
  - multipart form 字段：`file`。
  - 返回附件记录，其中 `access_key` 可作为 `file_key`。

## SOP

### 获取文件信息

1. 确认用户提供了 `file_key`。
2. 调用 `get_file(file_key=<file_key>)`。
3. 返回文件名、类型、大小和下载地址。
4. 如果用户要求读取内容，说明 MCP 不内联文件内容，并让用户或自动化代码使用 `download_url` 下载处理。

### 上传文件供评论使用

1. 确认本地文件路径和后续目标资源。
2. 使用 REST 附件接口上传文件。
3. 把返回的 `access_key` 作为 `file_key`。
4. 如果用户要添加评论，转交 `asp-comment-zh`，通过 `add_comment(..., file_keys=[file_key])` 关联附件。

## 输出规则

- 保持简洁。
- 不要声称已经读取文件内容，除非用户代码实际下载并解析了文件。
- 文件类型不受限制；不要假设一定是文本、图片或 JSON。

## 失败处理

- 如果 MCP 工具调用返回连接错误或超时，直接回复失败，提示用户检查 `ASP_MCP_URL`、`ASP_MCP_API_KEY`、ASGI `/api/mcp` 是否可访问，以及 API key 是否过期、用户是否被禁用。不要尝试重试或绕过。
- 如果 `file_key` 无效或文件不存在，直接说明。
