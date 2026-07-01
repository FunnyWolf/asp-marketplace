---
name: asp-file-en
description: 'Fetch file metadata and download URLs from ASP by file_key, and explain the attachment upload flow.'
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

Use this skill when the user wants to retrieve an ASP file download URL by `file_key`, or upload a file so it can be attached to a comment.

## When to Use

- The user has a `file_key` from a comment attachment or another ASP response and wants to download or inspect the file metadata.
- The user wants to upload a local file to ASP and use the returned key in a later comment.
- The user asks whether MCP can read comment attachment files.

## Operating Rules

- The unified MCP file entry point is `get_file(file_key)`.
- `get_file` returns metadata and a download URL only. It does not return file content, text, bytes, or base64.
- If file content must be parsed, state that user code or the client must download the file from `download_url` and process it.
- File upload is not an MCP tool. Use REST `POST /api/attachments/` with multipart field `file`; authentication still uses `Authorization: Api-Key <key>`.
- The returned `access_key` is the `file_key` used by later MCP tools.
- Do not invent `file_key` values or download URLs.

## MCP / REST Contract

- `get_file(file_key)`
  - Returns `file_key`, `filename`, `size`, `content_type`, and `download_url`.
  - If the file is missing, the call fails. Do not retry with guessed values.
- `POST /api/attachments/`
  - Multipart form field: `file`.
  - Returns an attachment record whose `access_key` can be used as `file_key`.

## SOP

### Get File Information

1. Confirm the user provided a `file_key`.
2. Call `get_file(file_key=<file_key>)`.
3. Return filename, type, size, and download URL.
4. If the user asks to read the content, explain that MCP does not inline file content and that user automation should download and process `download_url`.

### Upload a File for Comment Use

1. Confirm the local file path and later target resource.
2. Upload the file through the REST attachment endpoint.
3. Use the returned `access_key` as `file_key`.
4. If the user wants to add a comment, route to `asp-comment-en` and attach it through `add_comment(..., file_keys=[file_key])`.

## Output Rules

- Be concise.
- Do not claim to have read the file content unless user code actually downloaded and parsed it.
- File type is unrestricted. Do not assume it is text, an image, or JSON.

## Failure Handling

- If an MCP tool call returns a connection error or timeout, reply with failure immediately. Prompt the user to verify `ASP_MCP_URL`, `ASP_MCP_API_KEY`, that the ASGI `/api/mcp` endpoint is reachable, and that the API key is not expired and belongs to an active user. Do not retry or bypass.
- If `file_key` is invalid or the file does not exist, say so directly.
