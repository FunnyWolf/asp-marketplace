---
name: asp-module-creator-en
description: 'Create an ASP alert processing module. Use when the user wants to create an ASP module for a SIEM rule, write an alert processing script, or add a new Python module under the MODULES directory.'
argument-hint: '<rule-name>'
compatibility: connect to asp mcp server
disable-model-invocation: true
metadata:
  author: Funnywolf
  version: 0.1.0
  mcp-server: asp
  category: cyber security
  tags: [ module, siem, alert-processing, development ]
  documentation: https://asp.viperrtp.com/
---

# ASP Module Creator

Use this skill to guide the user through the full workflow — from requirement confirmation to code generation — when creating an ASP alert processing module for a SIEM rule.

## When to Use

- The user wants to create an ASP processing module for a SIEM rule.
- The user wants to add a new Python alert processing script under `MODULES/`.
- The user wants to integrate a SIEM alert into the ASP Alert/Case management pipeline.

## Operating Rules

- The module filename must exactly match the SIEM rule name (case-sensitive) — Rule name = Redis Stream name = filename. This is a hard constraint; any mismatch will prevent the framework from routing alerts to the module.
- A raw_alert sample must be obtained before writing any code. Never guess field structure.
- Before writing code, read the current backend enum models; enum values must come only from actual model definitions, never from memory or inference.
- All modules must inherit `BaseModule` and implement the `run()` method.
- SIRP data hierarchy: `Case → Alert → Artifact` (three-tier). Artifact is the smallest atomic investigation entity (an IP, a username); Alerts are attached to Cases; related alerts are aggregated into the same Case via `correlation_uid`. Enrichment is a cross-cutting attachment layer independent of the three-tier hierarchy — it can be attached to any level (Case / Alert / Artifact).
- Reference implementation: `backend/modules/aws_iam_privilege_escalation_attach_user_policy.py`, which demonstrates the current recommended pattern for consuming raw_alerts, extracting Artifacts, assembling Alert/Case records through `create_alert_with_context`, and requesting case analysis scheduling.

## Decision Flow

1. If the user has not provided a rule name, ask first.
2. If no raw_alert sample has been obtained, try the three methods in priority order (see SOP Step 3).
3. Analyse the field structure from the sample before writing code.
4. After generating the code, prompt the user to add the debug entry point and verify.

## SOP

### Step 1 — Get the Rule Name

Ask the user for the full SIEM Rule name, e.g. `XXX-01-YYY-ZZZ1-ZZZ2-ZZZ3`.
- The module file will be named `MODULES/XXX-01-YYY-ZZZ1-ZZZ2-ZZZ3.py`.
- Alerts will be consumed from the Redis Stream named `XXX-01-YYY-ZZZ1-ZZZ2-ZZZ3`.

### Step 2 — Confirm Prerequisites

Prompt the user to confirm all three of the following are ready:
1. A rule named `<rule-name>` exists in the SIEM.
2. The rule has already produced alerts.
3. Alerts have been forwarded to Redis Stream `<rule-name>` by the forwarding tool.

### Step 3 — Obtain a raw_alert Sample

Try the following methods in priority order; proceed as soon as one succeeds:

**Method A (recommended, requires ASP MCP connection):**
Call `read_stream_head(stream_name="<rule-name>", n=3)` to read the first few alerts from the stream.
Or call `read_stream_message_by_id(stream_name="<rule-name>", message_id=<id>)` to read a specific message.

**Method B (offline development):**
Ask the user to copy one or more raw_alert JSON samples to `DATA/MODULES/<rule-name>/raw_alert_*.json`, then read the file.

**Method C (direct paste):**
Ask the user to open Redis Insight, select the `<rule-name>` stream, copy a message's JSON content, and paste it into the conversation.

### Step 4 — Analyse the raw_alert Structure

Read the sample and identify:
- Event time field (e.g. `@timestamp`, `eventTime`)
- Principal identity fields (username, ARN, account ID, AccessKey, etc.)
- Target fields (target user, target resource, etc.)
- Network fields (source IP, User-Agent, etc.)
- Outcome fields (errorCode, outcome, status, etc.)
- Risk scoring fields (e.g. `event.risk_score`, `log.level`)
- Any other fields with investigation value

Before determining `correlation_uid`, first identify what kind of SOC scenario the rule describes. Different alert types require different aggregation logic. Do not mechanically reuse fixed fields or a fixed time window.

**Aggregation design goals:**
- One Case should represent one investigable, actionable security event or attack activity, not one log line and not an overly broad asset bucket.
- Prefer aggregation keys that are stable invariants of the attack activity. Avoid fields that vary by victim, session, request, or timestamp.
- Match the aggregation window to the response cadence: too short splits one activity and repeats notifications; too long delays response to a new wave.

**Think through aggregation keys in this order:**
1. Attacker dimension: source IP, sender, external account, malicious domain, malicious file hash, C2 domain, etc.
2. Target/victim dimension: target user, target host, target resource. Include these only when "same attacker against same target" is what defines one event.
3. Behaviour/payload dimension: email subject, URL domain, file hash, command-line signature, API name, rule subtype, etc. Include only when the field is stable and separates activities.
4. Environment dimension: cloud account, tenant, business system, region, etc. These are usually auxiliary keys and should not be the only aggregation key.

**Avoid using these as aggregation keys:**
- Random or high-cardinality fields: message_id, request_id, session_id, trace_id, uuid, exact timestamp.
- Victim fields when the scenario is broad delivery/scanning/brute force from the same attacker. Do not add every recipient/user/host to the key, or one campaign will fragment into many Cases.
- Overly broad fields: account_id, tenant_id, or rule_name alone will merge unrelated alerts.

**Common scenario guidance:**
- User-reported phishing mail: prefer sender/sender domain. If the email title does not contain random values such as recipient names, timestamps, or order numbers, include a normalized title. Usually do not include recipient. Recommended window: `12h`, which keeps one phishing wave together without delaying notifications and response too much.
- Same malicious URL/domain delivery: use URL domain or normalized URL + sender domain. If the URL contains one-time tokens, use only the domain or stable path.
- Malicious process/command on endpoint: use host + stable process/command signature. If it looks like lateral movement or a hash-wide outbreak, aggregate by file hash/command signature and do not necessarily include host.
- Cloud IAM abnormal operation: usually use cloud account/tenant + principal identity + API/target resource. If investigating one broad attack wave, aggregate by principal identity or source IP and keep target resources as supporting context. For high-risk permission changes such as `AttachUserPolicy`, if a Case represents "the same principal granting high-risk permissions to the same target user", use `account_id + principal_user/principal_id + target_user`; do not include fields such as `policyArn`, `requestID`, or `eventID` when they would fragment one activity or add little aggregation value.
- C2 communication: use C2 IP/domain + internal host. If one C2 affects many hosts, aggregate by C2 first and keep affected hosts in the Case.

**Time-window guidance:**
- User-reporting/notification-driven alerts: `6h`-`12h`, commonly `12h`.
- High-frequency scanning, brute force, C2 beaconing: `15m`-`2h`, adjusted by detection frequency and response need.
- Cloud permission changes, account anomalies, low-frequency high-risk operations: `4h`-`24h`.
- When using a window, explain the reason after generating the code.

If the aggregation key is unclear, propose candidate keys based on raw_alert and alert semantics, then ask the user to confirm. Do not default to `24h` or include every principal/target field without explaining why.

### Step 5 — Write the Module Code

**Prerequisite action:** read the current backend enum models (`apps.alerts.models`, `apps.artifacts.models`, `apps.cases.models`, `apps.enrichments.models`) and confirm all enum values that will be used before writing code.

Generate `MODULES/<rule-name>.py` using the following structure:

```python
from apps.agentic.runtime.base import BaseModule, parse_event_time, generate_correlation_uid
from apps.agentic.services.alerts import create_alert_with_context
from apps.alerts.models import (
    AlertAction,
    AlertAnalyticType,
    AlertPolicyType,
    AlertRiskLevel,
    AlertStatus,
    Confidence,
    Disposition,
    Impact,
    ProductCategory,
    Severity,
)
from apps.artifacts.models import ArtifactName, ArtifactRole, ArtifactType
from apps.cases.models import CaseConfidence, CaseImpact, CasePriority
from apps.enrichments.models import EnrichmentProvider, EnrichmentType


class Module(BaseModule):
    NAME = "<human-readable rule name>"
    DESC = "<short detection description>"
    STREAM_NAME = "<rule-name>"

    def run(self, message):
        # 1. Read raw alert
        raw_alert = message
        if not isinstance(raw_alert, dict):
            raise ValueError("Module expects a dict message.")

        # 2. Field extraction (customise based on raw_alert structure)
        event_time, time_unmapped = parse_event_time(raw_alert.get("@timestamp") or raw_alert.get("eventTime"))
        # ...

        # 3. Artifact extraction
        artifacts = []
        # artifacts.append({"type": ArtifactType.IP_ADDRESS, "role": ArtifactRole.ACTOR, "value": ..., "name": ArtifactName.SOURCE_IP})

        # 4. Compute correlation_uid
        # Choose keys and time window based on alert semantics; do not reuse fixed fields mechanically.
        correlation_uid = generate_correlation_uid(
            rule_id=self.STREAM_NAME,
            time_window=...,  # e.g. user-reported phishing mail can use "12h"
            keys=[...],  # stable attack-activity invariants; avoid random request/session IDs
            timestamp=event_time,
        )

        # 5. Create or reuse case, create alert, attach artifacts/enrichments, schedule analysis
        return create_alert_with_context(
            case_defaults={
                "title": ...,
                "severity": ...,
                "impact": ...,
                "priority": ...,
                "confidence": CaseConfidence.HIGH,
                "description": ...,
                "correlation_uid": correlation_uid,
                "tags": [...],
            },
            alert_fields={
                "title": ...,
                "severity": ...,
                "status": AlertStatus.NEW,
                "disposition": ...,
                "action": ...,
                "rule_id": self.STREAM_NAME,
                "rule_name": self.NAME,
                "correlation_uid": correlation_uid,
                "first_seen_time": event_time,
                "last_seen_time": event_time,
                "raw_data": raw_alert,
                "unmapped": {**time_unmapped, ...},
                # other fields...
            },
            artifacts=artifacts,
            enrichments=[],
            schedule_analysis=True,
            analysis_trigger=self.STREAM_NAME,
        )
```

Framework behaviour note:
- The framework continuously instantiates the Module class and calls `run()`. Each invocation processes exactly one alert — design the module to be stateless; do not accumulate cross-alert state in instance variables.

Field mapping principles:
- `alert_fields["raw_data"]`: store the full raw alert as a dict.
- `alert_fields["unmapped"]`: store fields that could not be mapped to Alert/Artifact standard fields.
- Alert field population priority: ① map directly from the raw alert; ② derive via calculation or transformation from raw alert fields; ③ use a sensible default only when both previous steps fail.
- MITRE ATT&CK fields (`tactic`, `technique`, `sub_technique`) should be hardcoded based on the alert type.
- `create_alert_with_context(...)` handles case lookup/create by `correlation_uid`, creates the alert, attaches artifacts and enrichments, and schedules analysis. Do not manually manage case-alert linking.
- Artifact selection principles: prefer stable, reusable, investigable entities; when deciding whether a field should become an Artifact, focus on whether it helps cross-alert correlation, investigation, evidence collection, or automated response.
- Avoid single-event random identifiers as Artifacts, such as request_id, event_id, trace_id, session_id, and uuid, unless the rule specifically investigates those IDs.
- Prefer preserving original, complete, stable entity values instead of only storing over-trimmed or overly generic derived values; derived values often fit better in Alert labels, descriptions, or unmapped data.
- ArtifactRole should describe the entity's relationship to the event: initiators are usually `ACTOR`, operated objects are usually `TARGET`, affected assets/environments are usually `AFFECTED`, and contextual entities are usually `RELATED` or `OTHER`.
- If fields in `unmapped` (or other high-value fields) need structured storage, add enrichment dictionaries to the `enrichments` argument for `create_alert_with_context(...)`.
- Use Enrichment only for supplementary information that is useful for investigation but does not fit core Alert/Case/Artifact fields. Do not convert every unmapped field into an Enrichment; keep it in `unmapped` by default and create Enrichment only when structured display or automation consumption is needed.
- For threat intelligence or owner attribution on an entity, prefer storing directly in artifact fields when supported; use enrichments only when richer structured content is needed.

### Step 6 — Create a Test Script

Create an independent test script outside the module file, for example under `backend/tests/` or another project-appropriate scratch path. Do **not** add `if __name__ == "__main__":` blocks to the module file itself — keep test code separate so the module stays clean.

Use the standard test script template:

```python
import os, sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parents[1] / "backend"))
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "asp.settings")
import django; django.setup()

import importlib.util
_mod = importlib.util.module_from_spec(
    importlib.util.spec_from_file_location("m", Path(__file__).resolve().parents[1] / "backend" / "modules" / "<module-file>.py")
)
_mod.__spec__.loader.exec_module(_mod)
Module = _mod.Module

if __name__ == "__main__":
    module = Module()
    # sample = {...}
    # module.run(sample)
```

Use one representative raw_alert sample per test run. Keep batch stream consumption in the framework, not in the module file.

## Clarification Rules

- If the user has not provided a rule name, ask before proceeding — never assume.
- If the MCP stream cannot be read, ask the user to choose Method B or Method C to obtain a sample.
- If the meaning of a raw_alert field is unclear, ask the user or consult relevant documentation before mapping.
- If the user has not specified correlation aggregation keys, infer them from the alert semantics and confirm with the user.

## Output Rules

- Generate complete, directly runnable Python file content.
- Keep code comments concise and in English, consistent with the `-en` convention.
- After generating the code, briefly explain the mapping logic for key fields so the user can review.
- Do not output content unrelated to the module code.
- Alert and Case `description` fields support markdown rendering, but are displayed as inline components in the frontend — avoid `#` `##` `###` heading syntax, `---` horizontal rules, HTML tags, and images, as they break the page layout. Other markdown features (bold, inline code, code blocks, lists, tables, links, blockquotes) are supported and should be used to improve readability.

## Failure Handling

- If an MCP tool call returns a connection error or timeout, reply with failure immediately. Prompt the user to verify `ASP_MCP_URL`, `ASP_MCP_API_KEY`, that the ASGI `/api/mcp` endpoint is reachable, and that the API key is not expired and belongs to an active user. Do not retry or bypass.
- If the ASP MCP cannot be connected and the user cannot provide a raw_alert sample, state that the workflow cannot continue and direct the user to complete the prerequisites first.
- If the raw_alert structure is abnormal (missing fields or excessively nested), describe the issue and ask the user to provide more samples or additional clarification.
- If the rule name provided by the user contains characters that are invalid in a Python filename, prompt the user to verify the name.
