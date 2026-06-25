---
name: asp-cmdb-en
description: 'Look up asset, identity, mailbox, resource, owner, and business context for ASP artifacts through CMDB providers.'
argument-hint: 'lookup cmdb <artifact_type> <artifact_value> [provider]'
compatibility: connect to asp mcp server
metadata:
  author: Funnywolf
  version: 0.1.0
  mcp-server: asp
  category: cyber security
  tags: [ cmdb, asset, identity, context, investigation ]
  documentation: https://asp.viperrtp.com/
---

# ASP CMDB Context

Use this skill when the user needs business, asset, identity, mailbox, resource, owner, or related context for an observable.

## When to Use

- The user wants to know what an IP, hostname, account, email, resource, subnet, port, or device belongs to.
- The user needs ownership, business service, asset criticality, or identity context before making a triage decision.
- An artifact investigation needs internal context in addition to SIEM evidence or threat intelligence.

## Operating Rules

- Use CMDB context to explain internal ownership and business meaning; do not treat it as threat intelligence.
- If the artifact already exists in ASP, first use the artifact value and type from the artifact record when available.
- If the user wants to persist useful CMDB context on a case, alert, or artifact, use `asp-enrichment-en`.

## MCP Tool Contract

- `cmdb_lookup(artifact_type, artifact_value, provider=None)`
  - `artifact_type` should be an ASP artifact type such as `IP Address`, `Hostname`, `User Name`, `Email Address`, `Resource UID`, `Port`, `Subnet`, `Account`, `Resource`, `Device`, or `Other`.
  - `artifact_value` is the exact value to look up.
  - `provider` is optional. Omit it to query all configured CMDB providers. If provided and unknown, the backend raises an error.
  - Returns `artifact_type`, `artifact_value`, per-provider `results`, and `errors`.
  - Each provider result may contain `asset`, `identity`, `mailbox`, `resource`, `port`, `subnet`, `business`, `owner`, `related`, `raw`, and `error`.

## SOP

1. Extract `artifact_type` and `artifact_value` from the request or current artifact context.
2. If the type is ambiguous, choose the most likely ASP artifact type and state the assumption.
3. Call `cmdb_lookup`.
4. Summarize only the context that changes investigation decisions.

Preferred response structure:

**CMDB Context**
- Observable: `<artifact_type> <artifact_value>`
- Provider(s)
- Owner / business service / criticality if present
- Asset or identity details if present
- Related objects if they matter

**Assessment**
- One short sentence explaining how the context affects triage or the next pivot.

## Clarification Rules

- Ask for `artifact_value` only when it is missing.
- Ask for `artifact_type` only when it cannot be reasonably inferred from the value or current artifact.
- Do not ask for `provider` unless the user explicitly wants a provider choice.

## Output Rules

- Be concise.
- Do not dump raw JSON unless the user asks for it.
- Separate observed CMDB facts from investigation judgment.

## Failure Handling

- If an MCP tool call returns a connection error or timeout, reply with failure immediately. Prompt the user to verify `ASP_MCP_URL`, `ASP_MCP_API_KEY`, that the ASGI `/api/mcp` endpoint is reachable, and that the API key is not expired and belongs to an active user. Do not retry or bypass.
- If the provider returns an error, report the error and suggest checking provider configuration.
- If no provider supports the artifact type, say so directly.
