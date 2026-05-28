---
name: asp-knowledge-en
description: 'Search and maintain ASP internal knowledge records. Supports keyword search and record updates.'
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

Use this skill when the user needs to retrieve or maintain internal knowledge on ASP.

## Design Rationale

Knowledge records are stored in the database with core fields: `title`, `body`, `tags`, `source`, `expires_at`.

- `title` — Knowledge title.
- `body` — Knowledge content, supports Markdown.
- `tags` — Labels for filtering and categorization.
- `source` — Origin: `Manual` for user-created, `Case` for auto-extracted from resolved Cases via the Knowledge Extraction playbook.
- `expires_at` — Expiration time. Empty means permanently valid. Expired records are excluded from search.

`search_knowledge` performs keyword matching against title and body, automatically filtering out expired records.

## When to Use

- The user wants to find relevant knowledge content by keyword.
- The user wants to update a knowledge record's title, body, tags, or expiration.

## Operating Rules

- If the user is looking for relevant knowledge content by topic, question, symptom, or natural-language query, prefer `search_knowledge`.
- If the user wants to modify a known knowledge record, use `update_knowledge`.

## Decision Flow

1. If the user wants to search knowledge content, use `search_knowledge`.
2. If the user wants to modify a known record, call `update_knowledge`.

## SOP

### Search Knowledge

1. Convert the user's question, keywords, or scenario description into a search query.
2. Call `search_knowledge`.
3. Return the most relevant few results, prioritizing directly relevant knowledge titles and short explanations.
4. Do not expand the full body unless the user explicitly asks for it.

Preferred response structure:

| Knowledge ID | Title | Tags |
|--------------|-------|------|

Then add one short line explaining how the results relate to the query.

### Update Knowledge

1. Require `knowledge_id`.
2. Extract only the fields the user explicitly wants to change: `title`, `body`, `expires_at`, `tags`.
3. Call `update_knowledge` with only the changed fields.
4. If the result is `None`, state that the knowledge record was not found.
5. Confirm only the fields that actually changed.

Preferred response structure:

- `Updated knowledge`: knowledge ID or returned row_id

## Clarification Rules

- Ask for `knowledge_id` only when the user wants to update a specific record and did not provide it.
- Ask one focused question only when the intent cannot be clearly mapped to `search_knowledge` or `update_knowledge`.

## Output Rules

- Be concise.
- Do not output the full knowledge body unless the user explicitly asks for it.
- When many records match, show the most valuable subset and briefly explain the overall pattern.

## Failure Handling

- If an MCP tool call returns a connection error or timeout, reply with failure immediately. Prompt the user to verify that the `ASP_MCP_SSE_URL` environment variable is configured and the ASP MCP server is running. Do not retry or bypass.
- If `search_knowledge` returns no results, say so directly and suggest a different keyword set.
- If the record to update does not exist, say so directly.
