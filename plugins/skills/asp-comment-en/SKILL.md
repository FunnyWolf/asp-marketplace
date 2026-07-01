---
name: asp-comment-en
description: 'Add comments to ASP records such as cases, alerts, artifacts, enrichments, knowledge records, or playbook runs, with attachments, replies, and @mentions.'
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

Use this skill when the user wants to add a natural-language note, analyst comment, handoff note, attachment comment, or read comment attachment metadata on an ASP record.

## When to Use

- The user says to add a note, comment, analyst observation, handoff note, or investigation remark.
- The content is human-readable narrative rather than structured enrichment data.
- The target is an existing case, alert, artifact, enrichment, knowledge record, or playbook run.
- The user wants to attach an already-uploaded file to a comment, or fetch a comment attachment download URL by `file_key`.

## Operating Rules

- Use `add_comment` for narrative notes, replies, @mentions, and attaching already-uploaded files.
- If the user provides a local file path instead of a `file_key`, first upload the file through REST `POST /api/attachments/` and use the returned `access_key` as the `file_key`.
- If the user wants to inspect or download a file, use `get_file(file_key)`; MCP returns file metadata and a download URL only, not file content.
- Use `asp-enrichment-en` instead when the user wants structured data, evidence, risk scores, IOC context, or machine-readable analysis saved.
- Do not invent target IDs, comment IDs, or file keys. If the target is ambiguous, ask for the exact record ID.

## MCP Tool Contract

- `add_comment(target_id, body="", file_keys=None, parent_id=None, mentions=None)`
  - `target_id` must start with one of: `case_`, `alert_`, `artifact_`, `enrichment_`, `knowledge_`, `playbook_`.
  - `body` may be empty only when at least one `file_key` is provided.
  - `file_keys` are attachment `access_key` / `file_key` values. They may be a single value, comma-separated string, JSON array string, or list.
  - `parent_id` replies to an existing comment and must belong to the same target resource.
  - `mentions` should use usernames, with numeric user ID compatibility.
  - The comment is authored as the authenticated MCP user.
- `get_file(file_key)`
  - `file_key` is the value returned on a comment attachment. It is the attachment `access_key`.
  - Returns `file_key`, `filename`, `size`, `content_type`, and `download_url`.
  - Does not return file body, bytes, base64, or parsed content.

## SOP

1. Identify the exact `target_id`.
2. Extract the comment body, attachments, reply target, and @mentions. Keep the user's wording when possible; tighten only obvious formatting.
3. If the user provides a local file path, upload it through the REST attachment endpoint first. If the user provides a `file_key`, use it directly.
4. Call `add_comment(target_id=<target_id>, body=<body>, file_keys=<file_keys>, parent_id=<parent_id>, mentions=<mentions>)`.
5. Confirm the comment was added and include the comment ID. If attachments were added, list their filenames and `file_key` values.

When reading a file:

1. Identify the `file_key`.
2. Call `get_file(file_key=<file_key>)`.
3. Return filename, type, size, and download URL. Do not claim to have read or parsed the file content.

Preferred response structure:

- `Target`: target ID
- `Comment`: created comment ID
- `Attachments`: filenames and `file_key` values, only when attachments are present

## Clarification Rules

- Ask for `target_id` if it is missing or ambiguous.
- Ask for comment text or a file to attach if both `body` and `file_key` / uploadable file are missing.
- Ask for the file source if the user says "this file" but provides neither a local path nor a `file_key`.
- If the user says "save this result" and the content is structured evidence, route to enrichment instead of comment.

## Output Rules

- Be concise.
- Do not repeat long comment bodies unless the user asks.

## Failure Handling

- If an MCP tool call returns a connection error or timeout, reply with failure immediately. Prompt the user to verify `ASP_MCP_URL`, `ASP_MCP_API_KEY`, that the ASGI `/api/mcp` endpoint is reachable, and that the API key is not expired and belongs to an active user. Do not retry or bypass.
- If the target record does not exist, say so directly.
- If `file_key`, `parent_id`, or `mentions` is invalid, say why the call failed; do not silently skip it.
