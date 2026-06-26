---
name: asp-siem-rule-en
description: 'Help users write Splunk SPL or ELK ES|QL detection rules. Use when users want to create SIEM detection rules, write alert queries, or design security detection logic.'
argument-hint: '<threat-scenario>'
compatibility: connect to asp mcp server
disable-model-invocation: true
metadata:
  author: Funnywolf
  version: 0.1.0
  mcp-server: asp
  category: cyber security
  tags: [ SIEM, detection, rule, SPL, ES|QL, MITRE ]
  documentation: https://asp.viperrtp.com/
---

# ASP SIEM Rule Creator

When users need to write SIEM detection rules, use this skill to guide them through the complete workflow from threat scenario description to deployable detection rules.

## When to Use

- Users want to create SIEM detection rules for a specific threat scenario.
- Users need to write Splunk SPL or ELK ES|QL alert queries.
- Users want to convert security detection logic into deployable SIEM rules.
- Users want to understand how to implement specific detection logic with SPL/ES|QL.

## Operating Rules

- Must confirm whether the user uses Splunk or ELK before proceeding; never assume.
- Must explore the schema before writing rules to understand available indices and fields.
- Must validate rules through SIEM API before outputting; never output unvalidated rules.
- MITRE ATT&CK mapping must be based on actual detection behavior; never fabricate mappings.
- Optimization goal is to generate directly deployable detection rules, not theoretical query examples.
- Rule validation requires a timezone-aware ISO 8601 time range because `siem_execute_spl` and `siem_execute_esql` require `time_range_start` and `time_range_end`.

## Decision Flow

1. Ask the user whether they use Splunk or ELK.
2. Call `siem_explore_schema()` to get available indices and help the user select the target index.
3. Call `siem_explore_schema(target_index=<index>)` to get field details.
4. Ask the user what threat scenario they want to detect.
5. Confirm a validation time range if it is missing.
6. Generate SPL or ES|QL detection rules based on the detection target and available fields.
7. Call `siem_execute_spl` or `siem_execute_esql` with `time_range_start` and `time_range_end` to validate the rule.
8. Adjust the rule based on validation results.
9. Map to MITRE ATT&CK tactics and techniques.
10. Output the final rule and usage instructions.

## MCP Tool Contract

- `siem_explore_schema(target_index=None)` lists indices or field metadata for one index.
- `siem_execute_spl(query, time_range_start, time_range_end, limit=100, time_field="@timestamp", index_name=None)` validates Splunk SPL.
- `siem_execute_esql(query, time_range_start, time_range_end, limit=100, time_field="@timestamp", index_name=None)` validates ELK ES|QL.
- Raw query validation requires `query`, `time_range_start`, and `time_range_end`; time values must be timezone-aware ISO 8601.

## SOP

### Step 1 — Confirm SIEM Type

Ask the user whether they use Splunk or ELK.

- If the user answers clearly, record the choice and continue.
- If the user is unsure, ask which platform they normally use to view alerts.
- Choosing Splunk will generate SPL rules; choosing ELK will generate ES|QL rules.

### Step 2 — Explore Schema

1. Call `siem_explore_schema()` to get the list of all indices.
2. Display available indices and help the user select the index relevant to their detection target.
3. Call `siem_explore_schema(target_index=<index>)` to get field details for the target index.
4. Summarize key field categories:
   - Time fields (e.g., `@timestamp`, `eventTime`)
   - Subject fields (username, IP, hostname, etc.)
   - Target fields (target user, target resource, etc.)
   - Result fields (status code, error message, etc.)
   - Network fields (source IP, destination IP, port, etc.)
5. Recommend field combinations suitable for detection.

### Step 3 — Confirm Detection Target

Ask the user what threat scenario they want to detect. Guide the user to clarify:

- What behavior to detect (e.g., brute force, abnormal login, privilege escalation, etc.)
- Key judgment conditions (e.g., failure count threshold, abnormal time range, etc.)
- Time window (e.g., past 1 hour, past 24 hours, etc.)

If the user's description is vague, guide with these questions:
- "What type of abnormal behavior do you want to detect?"
- "What are the key characteristics of this behavior?"
- "What time range do you want to detect within?"

### Step 4 — Generate Detection Rules

Generate detection rules based on the detection target and available fields:

**Splunk SPL Rule Structure:**
```
index=<index_name> <filter_conditions>
| stats count by <aggregation_fields>
| where count > <threshold>
```

**ELK ES|QL Rule Structure:**
```
FROM <index_name>
| WHERE <filter_conditions>
| STATS count = COUNT() BY <aggregation_fields>
| WHERE count > <threshold>
```

When generating rules, consider:
- Time filter conditions
- Field matching conditions
- Aggregation logic (e.g., by user, IP, host)
- Threshold settings
- Conditions to exclude false positives

### Step 5 — Validate Rules

1. Call `siem_execute_spl` or `siem_execute_esql` to execute the generated rule.
2. Analyze return results:
   - Whether there are matching records
   - Whether the match count is reasonable
   - Whether results match detection expectations
3. If no matches:
   - Check if the time range is appropriate
   - Check if field names are correct
   - Relax conditions and re-validate
4. If too many matches:
   - Add more filter conditions
   - Increase the threshold
   - Narrow the time range
5. Continue iterating until results are reasonable.

### Step 6 — MITRE ATT&CK Mapping

Map to MITRE ATT&CK based on the detected threat behavior:

- Identify which tactic the detected behavior belongs to (e.g., Initial Access, Execution, Persistence, etc.)
- Determine the specific technique (e.g., T1110 - Brute Force, T1078 - Valid Accounts, etc.)
- Output format: `Tactic: xxx | Technique: xxx (Txxxx)`

When mapping:
- Map based on actual detection behavior, do not guess
- One rule may map to multiple techniques
- If uncertain, explain the mapping rationale

### Step 7 — Output Final Rule

Output includes:

1. **Rule query statement**: Can be directly pasted into SIEM for use
2. **MITRE ATT&CK mapping**: Tactic and technique numbers
3. **Usage instructions**:
   - How to deploy to SIEM (e.g., Splunk saved search, ELK alerting rule)
   - Recommended execution frequency
   - Threshold adjustment suggestions
   - Possible false positive scenarios and exclusion methods

## Clarification Rules

- If the user hasn't specified the SIEM type, must ask first.
- If the user hasn't provided a detection target, must ask first.
- If the user is unsure about the target index, use `siem_explore_schema` to help select.
- If the user has questions about field meanings, explain based on sample values from schema results.
- If the user already has partial query statements, improve upon them rather than regenerating.

## Output Rules

- Rule code uses code block format for easy copying.
- Summarize each step's results concisely; do not list all fields.
- Validation results focus on match counts and representative records.
- Final rule must include complete executable query statements.
- Usage instructions should be concise and practical, not theoretical.

## Failure Handling

- If an MCP tool call returns a connection error or timeout, reply with failure immediately. Prompt the user to verify `ASP_MCP_URL`, `ASP_MCP_API_KEY`, that the ASGI `/api/mcp` endpoint is reachable, and that the API key is not expired and belongs to an active user. Do not retry or bypass.
- If `siem_explore_schema` returns an empty index list, explain that no indices are configured and prompt the user to complete index configuration first.
- If rule validation returns no results, check: whether the time range is appropriate, whether field names are correct, whether conditions are too strict.
- If rule validation returns too many results, suggest: adding more filter conditions, increasing the threshold, narrowing the time range.
- If the user's threat scenario cannot be detected with existing fields, explain the reason and suggest available alternatives.
