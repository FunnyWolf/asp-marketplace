---
name: asp-playbook-creator-en
description: 'Guide users in writing ASP playbook scripts under backend/playbooks, including LLM analysis playbooks and SOAR automation playbooks.'
argument-hint: '<playbook-goal>'
compatibility: backend playbook development
disable-model-invocation: true
metadata:
  author: Funnywolf
  version: 0.1.0
  category: cyber security
  tags: [ playbook, automation, soar, llm, development ]
  documentation: https://asp.viperrtp.com/
---

# ASP Playbook Creator

Use this skill when the user wants to create or modify an ASP playbook script. This skill only guides **playbook definition development**. It does not execute playbooks or inspect playbook runs; use `asp-playbook-en` for execution and run history.

## When to Use

- The user wants to create a new playbook script under `backend/playbooks/`.
- The user wants to write an LLM analysis, prompt-driven triage, or AI-assisted decision playbook.
- The user wants to write a SOAR automation playbook that calls internal services or external APIs.

## Runtime Contract

- File location: `backend/playbooks/<playbook_file>.py`; use snake_case for file names.
- Class definition: define `class Playbook(BasePlaybook)` and inherit from `apps.agentic.runtime.base.BasePlaybook`.
- Metadata: define `NAME`, `DESC`, and `TAGS`.
  - `NAME` is the executable playbook definition name. `execute_playbook(name=...)` matches this value exactly.
  - `DESC` explains the playbook's purpose.
  - `TAGS` classify the playbook, such as `["LLM", "Case"]` or `["SOAR", "Enrichment"]`.
- Entry point: implement `def run(self):` with no parameters.
- Runtime context:
  - `self.playbook_run`: the current playbook run record.
  - `self.case`: the Case linked to this run.
  - `self.user_input`: optional user guidance passed when the playbook is executed.
- Return value: `run()` returns a short string; the worker writes it to the playbook run `remark`.
- Failure handling: raise a clear `ValueError(...)` when required context is missing. Do not silently return success.

## Backend Execution Model

- `apps.agentic.services.playbooks` scans `settings.BASE_DIR / "playbooks"`, which is `backend/playbooks`.
- Discovery looks for a `Playbook` class inheriting from `BasePlaybook`.
- Worker flow: claim Pending run -> mark Running -> execute `Playbook(playbook_run=playbook_run).run()` -> write success `remark` or failure exception.
- Do not create `Playbook` run records inside playbook scripts.
- Do not write long-running loops inside playbooks. One `run()` handles the current run.

## Decision Flow

1. Confirm playbook type:
   - LLM analysis: custom analysis logic, prompts, triage flow, and optional write-back based on LLM output.
   - SOAR automation: internal service or external API calls for enrichment, notification, ticketing, blocking, isolation, or similar actions.
2. Confirm what Case data is needed: alerts, artifacts, comments, comment attachments, enrichments.
3. Confirm whether `user_input` should affect logic or prompts.
4. Confirm where output should go: run `remark` only, enrichment/comment write-back, or an external system.
5. For LLM analysis playbooks, confirm the prompt directory name. Store prompts under `custom/data/playbooks/<playbookname>/`.
6. Read related backend examples, models, and services before generating code.

## Reference Files

- `backend/apps/agentic/runtime/base.py`: `BasePlaybook`.
- `backend/apps/agentic/services/playbooks.py`: discovery, pending run creation, and status updates.
- `backend/apps/agentic/runtime/monitor.py`: worker execution of `run()`.
- `backend/playbooks/investigation.py`: LLM analysis example.
- `backend/playbooks/knowledge_extraction.py`: LLM knowledge extraction example.
- `backend/playbooks/threat_intelligence_enrichment.py`: SOAR enrichment example.
- `backend/custom/playbooks/cmdb_enrichment.py`: custom SOAR internal-service example.
- `backend/apps/agentic/analysis/prompts.py`: existing prompt-loading pattern.

## LLM Analysis Template

Use this pattern when the user wants custom analysis logic, prompt-driven triage, or automated decisions based on LLM output.

Store prompts under `custom/data/playbooks/<playbookname>/`, for example:

```text
backend/data/prompt/custom_triage/System_zh.md
backend/data/prompt/custom_triage/System_en.md
```

Use the project's prompt-language setting when loading prompt files:

```python
from pathlib import Path

from django.conf import settings
from apps.settings.runtime_config import get_prompt_language


def read_playbook_prompt(playbook_name, prompt_name="System"):
    filename = f"{prompt_name}_{get_prompt_language()}.md"
    return (Path(settings.BASE_DIR) / "data" / "prompt" / playbook_name / filename).read_text(encoding="utf-8")
```

Full LLM call example:

```python
import json
from pathlib import Path

from django.conf import settings
from langchain_core.messages import HumanMessage, SystemMessage
from pydantic import BaseModel, Field

from apps.agentic.runtime.base import BasePlaybook
from apps.settings.runtime_config import get_prompt_language
from integrations.llm.llmapi import LLMAPI


class CustomTriageLLMResult(BaseModel):
    summary: str
    verdict: str | None = None
    recommended_actions: list[str] = Field(default_factory=list)


def read_playbook_prompt(playbook_name, prompt_name="System"):
    filename = f"{prompt_name}_{get_prompt_language()}.md"
    return (Path(settings.BASE_DIR) / "data" / "prompt" / playbook_name / filename).read_text(encoding="utf-8")


def call_llm(system_prompt, payload):
    model = LLMAPI().get_model(tag="structured_output").with_structured_output(CustomTriageLLMResult)
    return model.invoke(
        [
            SystemMessage(content=system_prompt),
            HumanMessage(content=json.dumps(payload, ensure_ascii=False)),
        ]
    )


class Playbook(BasePlaybook):
    NAME = "<playbook name>"
    DESC = "<what this LLM analysis playbook does>"
    TAGS = ["LLM", "Case"]

    def run(self):
        if self.case is None:
            raise ValueError("<playbook name> requires a linked case.")

        case = self.case
        user_input = self.user_input

        # 1. Collect only the Case context this prompt needs.
        alerts = list(case.alerts.prefetch_related("artifacts", "enrichments"))
        artifacts = []
        for alert in alerts:
            artifacts.extend(alert.artifacts.all())

        # 2. Build prompt payload.
        system_prompt = read_playbook_prompt("custom_triage")
        user_prompt = {
            "case_id": case.case_id,
            "title": case.title,
            "severity": case.severity,
            "user_input": user_input,
            "artifacts": [
                {"artifact_id": item.artifact_id, "type": item.type, "value": item.value}
                for item in artifacts
            ],
        }

        # 3. Call the project LLM client. The structured_output tag selects a structured-output LLM config.
        result = call_llm(system_prompt, user_prompt)

        # 4. Optionally write enrichment/comment/fields, or only return run remark.
        return f"LLM analysis completed: {result.summary}"
```

Notes:
- `investigation.py` and `knowledge_extraction.py` only show how to use `self.case`, `self.user_input`, and `self.playbook_run`; do not assume their analysis logic must be reused.
- Prompt content is scenario-specific. Store it under `custom/data/playbooks/<playbookname>/`; do not hardcode long prompts in the playbook script.
- Use Enrichment for structured analysis results. Use Comment for analyst-readable natural-language notes.

## SOAR Automation Template

Use this pattern when the playbook calls internal services or external APIs for enrichment, notification, ticketing, blocking, isolation, or similar actions.

```python
from django.db import transaction

from apps.agentic.runtime.base import BasePlaybook
from apps.comments.services import create_record_comment
from apps.enrichments.models import Enrichment, EnrichmentProvider, EnrichmentType


@transaction.atomic
def upsert_artifact_enrichment(artifact, output):
    uid = f"custom:{artifact.artifact_id}"
    enrichment = (
        Enrichment.objects.select_for_update()
        .filter(
            artifact=artifact,
            provider=EnrichmentProvider.INTERNAL,
            type=EnrichmentType.OBSERVATION,
            uid=uid,
        )
        .first()
    )
    if enrichment is None:
        enrichment = Enrichment(
            artifact=artifact,
            provider=EnrichmentProvider.INTERNAL,
            type=EnrichmentType.OBSERVATION,
            uid=uid,
        )
    enrichment.name = "Automation Result"
    enrichment.value = artifact.value
    enrichment.desc = output.get("summary", "Automation completed.")
    enrichment.data = output
    enrichment.full_clean()
    enrichment.save()
    return enrichment


def add_case_comment(playbook_run, body):
    author = playbook_run.user if playbook_run and playbook_run.user_id else None
    if author is None:
        return None
    return create_record_comment(
        author=author,
        content_object=playbook_run.case,
        body=body,
    )


class Playbook(BasePlaybook):
    NAME = "<playbook name>"
    DESC = "<what this automation playbook does>"
    TAGS = ["SOAR", "Automation"]

    def run(self):
        if self.case is None:
            raise ValueError("<playbook name> requires a linked case.")

        stats = {
            "alerts": 0,
            "artifacts": 0,
            "processed": 0,
            "errors": 0,
        }

        alerts = list(self.case.alerts.prefetch_related("artifacts"))
        stats["alerts"] = len(alerts)

        for alert in alerts:
            for artifact in alert.artifacts.all():
                stats["artifacts"] += 1
                try:
                    # Call an internal service or external API.
                    output = {
                        "summary": f"Processed {artifact.type} {artifact.value}",
                        "artifact_id": artifact.artifact_id,
                    }
                    # output = your_service(artifact.type, artifact.value)

                    # Write structured output to artifact enrichment.
                    upsert_artifact_enrichment(artifact, output)
                    stats["processed"] += 1
                except Exception:
                    stats["errors"] += 1

        # Write a natural-language execution summary to the case.
        add_case_comment(
            self.playbook_run,
            f"{self.NAME} completed: processed={stats['processed']}, errors={stats['errors']}",
        )

        return (
            "<playbook name> completed. "
            f"alerts={stats['alerts']}, artifacts={stats['artifacts']}, "
            f"processed={stats['processed']}, errors={stats['errors']}"
        )
```

Related data access:
- Case: `self.case`
- Alerts: `self.case.alerts.all()` or `prefetch_related("artifacts", "enrichments")`
- Artifacts: `alert.artifacts.all()`
- Case enrichments: `self.case.enrichments.all()`
- Alert enrichments: `alert.enrichments.all()`
- Comments: query target object comments with `ContentType` + `Comment`.
- Comment attachments: use `comment.attachments.all()` and read file content with `attachment.file.open("rb")`.

Comment attachment example:

```python
from django.contrib.contenttypes.models import ContentType

from apps.comments.models import Comment


content_type = ContentType.objects.get_for_model(self.case, for_concrete_model=False)
comments = (
    Comment.objects
    .filter(content_type=content_type, object_id=str(self.case.pk))
    .prefetch_related("attachments")
)

for comment in comments:
    for attachment in comment.attachments.all():
        with attachment.file.open("rb") as file_obj:
            content = file_obj.read()
        filename = attachment.filename
```

Attachments are not restricted by file type. When generating a Playbook, do not assume an attachment is text; choose parsing logic based on filename, size, and actual content.

Write-back choices:
- Enrichment: structured results, enrichment data, external-system responses.
- Comment: natural-language notes, execution summaries, handoff information.
- Run remark: short execution summary returned by `run()`.

## SOP

### Step 1 — Confirm Playbook Type

Ask whether the user is writing:
- LLM analysis playbook
- SOAR automation playbook

### Step 2 — Confirm Inputs and Output

Confirm:
- What Case data is needed: alerts, artifacts, comments, comment attachments, enrichments.
- Whether `user_input` should guide logic or prompts.
- For LLM playbooks, the prompt directory name, such as `custom_triage`, and required prompt files.
- Whether output should be only `remark`, or also enrichment/comment/external-system write-back.

### Step 3 — Read References

Before generating code, inspect:
- `backend/apps/agentic/runtime/base.py`
- `backend/apps/agentic/services/playbooks.py`
- One similar `backend/playbooks/*.py` example

### Step 4 — Generate the Playbook File

Generate complete content for `backend/playbooks/<playbook_file>.py`. It must include:
- `class Playbook(BasePlaybook)`
- `NAME`, `DESC`, `TAGS`
- `def run(self):`
- Required `self.case` checks
- A short, readable return remark

### Step 5 — Explain Registration and Execution

After generating code, remind the user:
- `NAME` is the definition name used by `execute_playbook(name=...)`.
- If the worker is already running, it may need restart/reload before discovering the new file.
- Use `list_playbook_templates()` to confirm discovery.

## Clarification Rules

- If the playbook type is missing, ask whether it is LLM analysis or SOAR automation.
- If target Case data needs are unclear, read the minimum needed context; do not default to loading everything.
- If an external API is requested but endpoint, auth, or parameters are missing, ask for the missing item.
- If LLM analysis is requested but the analysis goal is unclear, ask what question the playbook should answer or what output shape is expected.

## Output Rules

- Generate complete runnable Python file content.
- Keep code comments concise.
- Do not mix playbook execution MCP operations into the creator workflow; execution belongs to `asp-playbook-en`.
- Do not invent internal services. Search backend for existing services/helpers before using one.

## Failure Handling

- If playbook type is unclear, ask one focused question.
- If Case context is required, generated code must explicitly `raise ValueError(...)` when missing.
- If external API details are missing, do not generate fake endpoints or tokens.
