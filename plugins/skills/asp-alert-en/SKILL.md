---
name: asp-alert-en
description: 'Review ASP alerts for triage analysis.'
argument-hint: 'review alert <alert_id> | list alerts [filters]'
compatibility: connect to asp mcp server
metadata:
  author: Funnywolf
  version: 0.1.1
  mcp-server: asp
  category: cyber security
  tags: [ alert-management, soc, triage, investigation ]
  documentation: https://asp.viperrtp.com/
---

# ASP Alert

Use this skill when the user needs to work on ASP alerts for SOC analysis.
An alert is secondary data in ASP. Each alert belongs to a case, and each alert can have one or more artifacts attached.

## When to Use

- The user gives an alert ID and wants a quick review, inspection, or summary.
- The user wants to find alerts by status, severity, confidence, or correlation UID.
- The user wants to attach enrichment to an alert after analysis.

## Operating Rules

- Keep the response focused on triage value rather than repeating schema fields.
- If the user is working on a specific alert, prefer `list_alerts(alert_id=<id>, limit=1, include_related=True, include_comments=True)` because the current MCP surface does not expose a separate `get_alert` tool.
- Alerts are currently read-only. If the user wants to save structured analysis back onto the alert, use the `asp-enrichment-en` skill.
- If the user wants a natural-language note on the alert, use the `asp-comment-en` skill.

## Additional Information

- `alert_id` is the MCP-facing record identifier for each alert, for example `alert_000001`.

## Decision Flow

1. If the user provides a specific alert ID or says "open", "show", "review", or "summarize" an alert, call `list_alerts(alert_id=<id>, limit=1, include_related=True, include_comments=True)`.
2. If the user wants to browse or compare alerts, use `list_alerts(..., include_related=False, include_comments=False)` with supported filters.
3. If the user wants to attach analysis results, intelligence, or structured context to the alert, use the `asp-enrichment-en` skill.

## MCP Tool Contract

- `list_alerts(alert_id=None, status=None, severity=None, confidence=None, correlation_uid=None, include_related=False, limit=10, include_comments=False, comments_limit=20)`
  - `alert_id` is a readable ID such as `alert_000001`.
  - `status`: `Unknown`, `New`, `In Progress`, `Suppressed`, `Resolved`, `Archived`, `Deleted`, `Other`.
  - `severity`: `Unknown`, `Informational`, `Low`, `Medium`, `High`, `Critical`.
  - `confidence`: `Unknown`, `Low`, `Medium`, `High`.
  - `correlation_uid` matches alerts linked to the same case correlation.
  - `limit` is clamped to 1-100.
  - `include_related=False` returns a compact alert record. Set `include_related=True` only when reviewing a specific alert and you need artifacts and enrichments.
  - `include_comments=True` includes recent comments. `comments_limit` defaults to 20 and is capped at 50.
  - Comment attachments include metadata and download URLs only. Use `asp-file-en` / `get_file` to fetch file metadata again.

## SOP

### Review One Alert

1. If the user wants to review, analyze, or inspect alert details, call `list_alerts(alert_id=<id>, limit=1, include_related=True, include_comments=True)`.
2. If the user only needs the basic alert information, call `list_alerts(alert_id=<id>, limit=1, include_related=False, include_comments=False)`.
3. If the result is empty, state that the alert was not found.
4. Parse the first JSON record.
5. Present only the most useful triage fields.

Preferred response structure:

- `Alert`: alert ID, title or name, severity, status, confidence, correlation UID.
- `Timeline`: created time.
- `Key Context`: source UID, rule ID, rule name, correlation UID, linked case ID, artifacts, and enrichments when relevant.
- `Assessment`: short triage judgment.

### List Alerts

1. Extract supported filters: `alert_id`, `status`, `severity`, `confidence`, `correlation_uid`, `limit`.
2. Normalize natural-language filters before calling MCP.
3. Call `list_alerts`.
4. Parse the returned JSON strings.
5. Present a compact comparison view.

Preferred response structure:

| Alert ID | Title | Severity | Status | Confidence | Created | Rule Name |
|----------|-------|----------|--------|------------|---------|-----------|

Then add one short explanation line when needed.

## Clarification Rules

- Ask for `alert_id` only when it is missing for alert-related actions.
- Ask for enum clarification only when the requested value does not map cleanly to ASP values.

## Output Rules

- Be concise.
- Do not output raw JSON unless the user explicitly asks for it.
- Prefer triage wording over schema wording.
- State blockers clearly: alert not found, unsupported filter, invalid enum value.

## Failure Handling

- If an MCP tool call returns a connection error or timeout, reply with failure immediately. Prompt the user to verify `ASP_MCP_URL`, `ASP_MCP_API_KEY`, that the ASGI `/api/mcp` endpoint is reachable, and that the API key is not expired and belongs to an active user. Do not retry or bypass.
- If the alert does not exist, say so directly.
- If filters return no results, say so directly and suggest the most useful refinement.
