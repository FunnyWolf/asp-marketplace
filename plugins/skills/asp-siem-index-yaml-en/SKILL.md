---
name: asp-siem-index-yaml-en
description: 'Create or update SIEM index configuration YAML. Use when the user wants to generate field configs for a SIEM index, refresh an existing index YAML, or sync live backend fields into a config file.'
argument-hint: '<index_name> <backend>'
compatibility: connect to asp mcp server
metadata:
  author: Funnywolf
  version: 0.1.0
  mcp-server: asp
  category: cyber security
  tags: [ SIEM, index, yaml, schema, configuration ]
  documentation: https://asp.viperrtp.com/
---

# ASP SIEM Index YAML

Use this skill when the user wants to create or update a SIEM index configuration YAML, guiding the full workflow from field discovery to writing the config file.

## When to Use

- User wants to generate an index configuration YAML for a SIEM index.
- User wants to update an existing index YAML with live backend fields.
- User wants to see what fields actually exist in an ELK or Splunk index.

## Rules

- Config files are stored at `DATA/PLUGINS/SIEM/<index_name>.yaml`.
- Must call `siem_discover_index_fields` to fetch live fields from the backend — never skip this and write fields manually.
- `name`, `type`, and `sample_values` are taken directly from discovery results. `sample_values` must preserve native types — do not cast everything to strings.
- `description` and `is_key_field` are inferred by the model based on field semantics and sample_values, marked as pending confirmation.
- Do not overwrite an existing YAML before the user confirms.

## Decision Flow

1. If `index_name`, `backend`, or the sampling time range is missing, ask the user first.
2. If `DATA//Plugin/SIEM/<index_name>.yaml` already exists, read it as a baseline for comparison.
3. Call `siem_discover_index_fields` to get live fields.
4. Generate a draft and present it for user review.
5. Write the file only after user confirmation.

## SOP

### Step 1 — Gather Input

Ask the user for:
- `index_name`: the SIEM index name.
- `backend`: `ELK` or `Splunk`.
- `time_range_start` and `time_range_end`: UTC or timezone-aware ISO 8601 sampling range.

### Step 2 — Check Existing Config

Check whether `DATA/PLUGINS/SIEM/<index_name>.yaml` already exists.
- If it exists, read it as a baseline and show diffs later.
- If not, mark as new creation.

### Step 3 — Discover Fields

Call `siem_discover_index_fields(index_name=<index_name>, backend=<backend>, time_range_start=<start>, time_range_end=<end>, max_samples_per_field=20)`.

MCP tool contract:
- `siem_discover_index_fields(index_name, backend, time_range_start, time_range_end, doc_limit=10000, max_samples_per_field=20)`
  - `backend` must be `ELK` or `Splunk`.
  - `time_range_start` and `time_range_end` are required and must be timezone-aware ISO 8601, for example `2026-06-23T12:00:00Z`.
  - `doc_limit` is clamped by the backend to 1-100000.
  - `max_samples_per_field` is clamped by the backend to 1-100.

Optional parameters:
- `doc_limit`: number of documents to scan, default 10000. Lower values return faster but may miss sample values; higher values improve coverage but take longer.
- `max_samples_per_field`: max sample values returned per field, default 20. Keep the default — the adaptive rules below will trim as needed.

The response contains for each field:
- `name`: field name (nested fields use dot-path notation)
- `type`: field type as reported by the backend
- `sample_values`: sample value list (native types preserved, up to 20)

**Adaptive sample count rules:**

Determine the appropriate number of `sample_values` per field based on its characteristics:

| Field characteristic | Sample count | How to identify |
|----------------------|-------------|-----------------|
| Timestamp (date, timestamp, datetime) | 1 | Type is date/timestamp, or values match ISO-8601 / Unix epoch patterns |
| High-cardinality long text (UUID, hash, URL, path, raw log) | 1-2 | Value length > 32 chars, or field name contains uuid/hash/url/path/uri/raw/line |
| Boolean / status flag | Up to 2 | Type is boolean, or values are only true/false/0/1 binary |
| Low-cardinality enum (≤ 10 distinct values) | All | Deduplicated value count ≤ 10, list them all |
| Mid-cardinality enum (11-30 distinct values) | 10 | Take top-10 by frequency, note "N distinct values" in description |
| High-cardinality enum (> 30 distinct values) | 5 | Take top-5 by frequency, note "N distinct values, top-5 listed" in description |
| Other (plain text, numbers) | 5 | Default top-5 |

The backend may return a `sample_values` count that does not match the rules above. Handle as follows:
- If the backend returns **fewer** than required: keep as-is, do not pad.
- If the backend returns **more** than required: trim per the rules above, and note the trimming reason in the draft.
- If the backend returns **exactly** the right amount: take directly.

### Step 4 — Generate Draft

Populate the full config for each field:

| Field | Source |
|-------|--------|
| `name` | Taken directly |
| `type` | Taken directly |
| `sample_values` | Trimmed per adaptive rules above; if empty list, note "(no sample data, description is inference-only)" in description |
| `description` | Generated from field name semantics, enriched with sample_values range; for enum types, note the total distinct value count |
| `is_key_field` | Heuristic based on investigation value: identity, asset, network tuple, action, outcome, high-signal identifier → `true`; noise or low-value fields → `false` |

### Step 5 — Present Draft

Show the user a diff summary and the full draft.

Preferred reply structure:

**Draft Summary**

- Target index: `<index_name>`
- Backend: `<backend>`
- Total fields: `<n>`
- New fields: `<list>` (compared to baseline, if any)
- Type changes: `<list>` (compared to baseline, if any)
- Fields with `is_key_field=true`

**Pending Confirmation**

- Whether `is_key_field` inferences are reasonable
- Whether `description` values match naming conventions
- Whether any fields need adjustment

### Step 6 — Write File

After user confirmation, write the full YAML to `DATA/PLUGINS/SIEM/<index_name>.yaml`.

Top-level YAML structure:

```yaml
name: <index_name>
backend: <backend>
description: <index description>

fields:
  - name: <field_name>
    type: <field_type>
    description: <field_description>
    is_key_field: <true|false>
    sample_values: [<value1>, <value2>, ...]
```

## Clarification Rules

- Only ask for `index_name`, `backend`, or sampling time range when missing.
- If `siem_discover_index_fields` returns an empty field list, the index name may be wrong or the backend has no data — ask the user to verify.
- If the user disagrees with `is_key_field` or `description` for certain fields, adjust per their request and re-present.

## Output Rules

- When presenting the draft, prefer a diff summary + key field list over dumping the full YAML.
- Only produce the full YAML file content when the user confirms the write.
- Keep it concise — do not re-explain each field's meaning.

## Failure Handling

- If an MCP tool call returns a connection error or timeout, reply with failure immediately. Prompt the user to verify `ASP_MCP_URL`, `ASP_MCP_API_KEY`, that the ASGI `/api/mcp` endpoint is reachable, and that the API key is not expired and belongs to an active user. Do not retry or bypass.
- If `siem_discover_index_fields` fails, report the error and ask the user to check the index name and backend connectivity.
- If the returned field count is 0, prompt the user to verify the index exists and contains data.
- If the user requests a write without having reviewed the draft, remind them to complete the review step first.
