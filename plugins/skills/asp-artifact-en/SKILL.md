---
name: asp-artifact-en
description: 'Find artifacts by IOC.'
argument-hint: 'review artifact <artifact_id> | list artifacts [filters]'
compatibility: connect to asp mcp server
metadata:
  author: Funnywolf
  version: 0.1.1
  mcp-server: asp
  category: cyber security
  tags: [ artifact, pivot, enrichment, investigation ]
  documentation: https://asp.viperrtp.com/
---

# ASP Artifact

Use this skill when the user needs to investigate artifacts on ASP.
Artifacts are created automatically by system processes. You can list and analyze existing artifacts, and save analysis back through enrichment.

## When to Use

- The user wants to find artifacts by value, type, role, or artifact ID.

## Operating Rules

- Treat artifacts as the smallest investigation object on the platform.
- Use `list_artifacts` for lookup and review.
- If the user wants to save analysis on the artifact, use the `asp-enrichment-en` skill.

## Additional Information

- `artifact_id` is the MCP-facing record identifier for each artifact, for example `artifact_000001`.

## Decision Flow

1. If the user wants to find or review one specific artifact, call `list_artifacts(artifact_id=<id>, limit=1, include_related=True, include_comments=True)`.
2. If the user wants to browse or compare artifacts, call `list_artifacts(..., include_related=False, include_comments=False)`.
3. If the user wants to attach intelligence, analysis notes, or structured analysis to an artifact, use the `asp-enrichment-en` skill.
4. If the user is investigating from an artifact, treat the artifact as the smallest pivot object and suggest the next most useful hop only when needed.

## MCP Tool Contract

- `list_artifacts(artifact_id=None, type=None, role=None, value=None, include_related=False, limit=10, include_comments=False, comments_limit=20)`
  - `artifact_id` is a readable ID such as `artifact_000001`.
  - `type` accepts ASP artifact type values such as `IP Address`, `Hostname`, `User Name`, `Email Address`, `URL String`, `Hash`, `File Name`, `Process Name`, `Resource UID`, `Port`, `Subnet`, `Command Line`, `CVE`, `Account`, `Resource`, `File Path`, `Device`, `Registry Path`, `Other`, and related platform values.
  - `role`: `Unknown`, `Target`, `Actor`, `Affected`, `Related`, `Other`.
  - `value` is an exact artifact value match.
  - `type` and `role` may be strings, comma-separated strings, JSON array strings, or lists.
  - `include_related=False` returns compact artifact records. Set `include_related=True` only when reviewing a specific artifact and you need enrichments.
  - `include_comments=True` includes recent comments. `comments_limit` defaults to 20 and is capped at 50.
  - Comment attachments include metadata and download URLs only. Use `asp-file-en` / `get_file` to fetch file metadata again.
  - `limit` is clamped to 1-100.
  - There is no owner filter in the current MCP tool.

## SOP

### List Artifacts

1. Extract the narrowest useful filters from the request.
2. Use `include_related=False, include_comments=False` for broad lookup/listing. Use `include_related=True` only for one specific artifact that needs enrichment context, and `include_comments=True` only when comments are needed.
3. Call `list_artifacts`.
4. Parse the returned JSON strings.
5. Present a compact artifact view. If the user will likely attach or reuse the artifact next, surface the `artifact_id` explicitly.

Preferred response structure:

| Artifact ID | Value | Type | Role | Summary |
|-------------|-------|------|------|---------|

Then add one short explanation line when needed.

## Clarification Rules

- Ask for `artifact_id` only when the user wants to enrich an existing artifact and did not provide it.

## Output Rules

- Be concise.
- Do not output raw JSON unless the user explicitly asks for it.
- Prefer pivot-oriented wording over storage wording.
- When many artifacts match, show the most valuable subset and briefly explain the overall pattern.

## Failure Handling

- If an MCP tool call returns a connection error or timeout, reply with failure immediately. Prompt the user to verify `ASP_MCP_URL`, `ASP_MCP_API_KEY`, that the ASGI `/api/mcp` endpoint is reachable, and that the API key is not expired and belongs to an active user. Do not retry or bypass.
- If no artifacts match, say so directly and suggest the most useful refinement.
- If the target artifact does not exist, say so directly.
