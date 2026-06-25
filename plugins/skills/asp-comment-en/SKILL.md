---
name: asp-comment-en
description: 'Add natural-language comments to ASP records such as cases, alerts, artifacts, enrichments, knowledge records, or playbook runs.'
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

Use this skill when the user wants to add a natural-language note or analyst comment to an ASP record.

## When to Use

- The user says to add a note, comment, analyst observation, handoff note, or investigation remark.
- The content is human-readable narrative rather than structured enrichment data.
- The target is an existing case, alert, artifact, enrichment, knowledge record, or playbook run.

## Operating Rules

- Use `add_comment` for narrative notes.
- Use `asp-enrichment-en` instead when the user wants structured data, evidence, risk scores, IOC context, or machine-readable analysis saved.
- Do not invent target IDs. If the target is ambiguous, ask for the exact record ID.

## MCP Tool Contract

- `add_comment(target_id, body)`
  - `target_id` must start with one of: `case_`, `alert_`, `artifact_`, `enrichment_`, `knowledge_`, `playbook_`.
  - `body` is required and must not be empty.
  - The comment is authored as the authenticated MCP user.
  - Returns the created comment `id`, `body`, `author`, and `created_at`.

## SOP

1. Identify the exact `target_id`.
2. Extract the comment body. Keep the user's wording when possible; tighten only obvious formatting.
3. Call `add_comment(target_id=<target_id>, body=<body>)`.
4. Confirm the comment was added and include the comment ID.

Preferred response structure:

- `Target`: target ID
- `Comment`: created comment ID

## Clarification Rules

- Ask for `target_id` if it is missing or ambiguous.
- Ask for the comment body if it is missing.
- If the user says "save this result" and the content is structured evidence, route to enrichment instead of comment.

## Output Rules

- Be concise.
- Do not repeat long comment bodies unless the user asks.

## Failure Handling

- If an MCP tool call returns a connection error or timeout, reply with failure immediately. Prompt the user to verify `ASP_MCP_URL`, `ASP_MCP_API_KEY`, that the ASGI `/api/mcp` endpoint is reachable, and that the API key is not expired and belongs to an active user. Do not retry or bypass.
- If the target record does not exist, say so directly.
- If the body is empty, ask for the comment text.
