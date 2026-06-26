---
name: asp-playbook-creator-zh
description: '指导编写 ASP playbook 脚本。适用于在 backend/playbooks 下创建 LLM 智能分析类 playbook 或 SOAR 自动化处理类 playbook。'
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

当用户想新建或修改 ASP playbook 脚本时，使用这个 skill。它只指导**编写 playbook definition**，不负责执行 playbook 或查询 playbook run；执行和查询仍使用 `asp-playbook-zh`。

## 适用场景

- 用户要在 `backend/playbooks/` 下新建 playbook 脚本。
- 用户要编写 LLM 智能分析、提示词研判、自动分诊类 playbook。
- 用户要编写调用内部服务或外部接口的 SOAR 自动化处理 playbook。

## 运行契约

- 文件位置：`backend/playbooks/<playbook_file>.py`，文件名建议使用 snake_case。
- 类定义：必须定义 `class Playbook(BasePlaybook)`，继承 `apps.agentic.runtime.base.BasePlaybook`。
- 元数据：必须定义 `NAME`、`DESC`、`TAGS`。
  - `NAME` 是可执行的 playbook definition 名称，`execute_playbook(name=...)` 会精确匹配它。
  - `DESC` 用于解释该 playbook 的目的。
  - `TAGS` 用于分类，例如 `["LLM", "Case"]`、`["SOAR", "Enrichment"]`。
- 入口方法：实现 `def run(self):`，不接收参数。
- 上下文：
  - `self.playbook_run`：当前 playbook run 记录。
  - `self.case`：当前 run 关联的 Case。
  - `self.user_input`：用户在执行 playbook 时传入的补充说明。
- 返回值：`run()` 返回简短字符串，worker 会把它写入 playbook run 的 `remark`。
- 失败处理：缺少必要上下文时显式 `raise ValueError(...)`，不要静默返回成功。

## 后端运行机制

- `apps.agentic.services.playbooks` 扫描 `settings.BASE_DIR / "playbooks"`，即 `backend/playbooks`。
- 扫描器查找继承 `BasePlaybook` 的 `Playbook` 类。
- worker 流程：认领 Pending run → 标记 Running → 执行 `Playbook(playbook_run=playbook_run).run()` → 成功写入 `remark` → 失败写入异常信息。
- 不要在 playbook 脚本中自己创建 `Playbook` run 记录。
- 不要把 playbook 写成长期循环任务；一次 `run()` 只处理当前 run。

## 决策流程

1. 先确认 playbook 类型：
   - LLM 智能分析类：自定义分析逻辑、提示词、研判流程，可能根据输出继续分诊或写回。
   - SOAR 自动化处理类：调用内部服务或外部接口，完成富化、通知、工单、封禁、隔离等动作。
2. 确认目标 Case 数据需求：是否需要 alerts、artifacts、comments、enrichments。
3. 确认是否需要 `user_input` 影响逻辑或 prompt。
4. 确认输出写到哪里：只写 run `remark`，还是额外创建 enrichment/comment，或调用外部系统。
5. LLM 智能分析类还要确认提示词目录名，推荐放在 `backend/data/prompt/<playbookname>/`。
6. 读取相关 backend 示例和模型/服务后再生成代码。

## 参考文件

- `backend/apps/agentic/runtime/base.py`：`BasePlaybook` 定义。
- `backend/apps/agentic/services/playbooks.py`：playbook discovery、pending run 创建和状态更新。
- `backend/apps/agentic/runtime/monitor.py`：worker 如何执行 `run()`。
- `backend/playbooks/investigation.py`：LLM 分析类基础示例。
- `backend/playbooks/knowledge_extraction.py`：LLM 知识提取类基础示例。
- `backend/playbooks/threat_intelligence_enrichment.py`：SOAR 富化类示例。
- `backend/playbooks/cmdb_enrichment.py`：SOAR 内部服务调用类示例。
- `backend/apps/agentic/analysis/prompts.py`：现有提示词加载方式示例。

## LLM 智能分析类模板

适用于用户希望编写自己的分析逻辑、提示词、研判流程，或根据 LLM 输出继续做自动化分诊/写回的场景。

提示词建议放在 `backend/data/prompt/<playbookname>/` 下，例如：

```text
backend/data/prompt/custom_triage/System_zh.md
backend/data/prompt/custom_triage/System_en.md
```

加载提示词可复用项目现有语言配置：

```python
from pathlib import Path

from django.conf import settings
from apps.settings.runtime_config import get_prompt_language


def read_playbook_prompt(playbook_name, prompt_name="System"):
    filename = f"{prompt_name}_{get_prompt_language()}.md"
    return (Path(settings.BASE_DIR) / "data" / "prompt" / playbook_name / filename).read_text(encoding="utf-8")
```

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

        # 1. 收集上下文：按需读取 case、alerts、artifacts、comments、enrichments。
        alerts = list(case.alerts.prefetch_related("artifacts", "enrichments"))
        artifacts = []
        for alert in alerts:
            artifacts.extend(alert.artifacts.all())

        # 2. 构造你自己的 prompt / input。
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

        # 3. 调用项目 LLM client。structured_output tag 会选择支持结构化输出的 LLM 配置。
        result = call_llm(system_prompt, user_prompt)

        # 4. 按需要写回 enrichment/comment/字段，或只返回 run remark。
        return f"LLM analysis completed: {result.summary}"
```

注意：
- `investigation.py` 和 `knowledge_extraction.py` 只展示如何使用 `self.case`、`self.user_input`、`self.playbook_run`；不要求复用它们的分析逻辑。
- Prompt 内容由用户按场景设计，推荐放在 `backend/data/prompt/<playbookname>/`，不要硬编码在 playbook 脚本里。
- 如果要保存结构化分析结果，优先写 Enrichment；如果只是分析师可读说明，可写 Comment。

## SOAR 自动化处理类模板

适用于调用内部服务或外部接口完成富化、通知、工单、封禁、隔离等动作。

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
                    # 调用内部服务或外部接口。
                    output = {
                        "summary": f"Processed {artifact.type} {artifact.value}",
                        "artifact_id": artifact.artifact_id,
                    }
                    # output = your_service(artifact.type, artifact.value)

                    # 写入结构化结果到 artifact enrichment。
                    upsert_artifact_enrichment(artifact, output)
                    stats["processed"] += 1
                except Exception:
                    stats["errors"] += 1

        # 写入自然语言评论到 case，适合记录自动化执行摘要。
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

关联数据获取：
- Case：`self.case`
- Alerts：`self.case.alerts.all()` 或 `prefetch_related("artifacts", "enrichments")`
- Artifacts：`alert.artifacts.all()`
- Case enrichments：`self.case.enrichments.all()`
- Alert enrichments：`alert.enrichments.all()`

写回方式简述：
- Enrichment：适合结构化结果、富化数据、外部系统返回。
- Comment：适合自然语言评论、执行说明、交接信息。
- Run remark：适合简短执行摘要，由 `run()` 返回字符串即可。

## SOP

### Step 1 — 确认 Playbook 类型

询问用户要写哪类 playbook：
- LLM 智能分析类
- SOAR 自动化处理类

### Step 2 — 确认输入与目标

确认：
- 目标 case 数据需要哪些部分：alerts、artifacts、comments、enrichments。
- 是否需要用户通过 `user_input` 提供额外指导。
- LLM 智能分析类的提示词目录名，例如 `custom_triage`，以及需要哪些 prompt 文件。
- 输出只写 `remark`，还是还要写 enrichment/comment 或调用外部系统。

### Step 3 — 读取参考实现

生成代码前，至少查看：
- `backend/apps/agentic/runtime/base.py`
- `backend/apps/agentic/services/playbooks.py`
- 一个同类型 `backend/playbooks/*.py` 示例

### Step 4 — 生成 Playbook 文件

生成完整 `backend/playbooks/<playbook_file>.py` 内容，必须包含：
- `class Playbook(BasePlaybook)`
- `NAME`、`DESC`、`TAGS`
- `def run(self):`
- 必要的 `self.case` 检查
- 简短、可读的返回 remark

### Step 5 — 说明注册与执行

生成后提醒：
- `NAME` 就是 `execute_playbook(name=...)` 使用的 definition 名称。
- 如果 worker 已运行，可能需要重启或重新加载后才能发现新文件。
- 可通过 `list_playbook_templates()` 检查新 playbook 是否被发现。

## 澄清规则

- 用户没说明 playbook 类型时，先问 LLM 分析还是 SOAR 自动化。
- 用户没说明目标 case 数据需求时，按最小需要读取，避免默认拉取全部上下文。
- 用户要求外部接口调用但没有 endpoint、认证方式或参数时，先问缺失项。
- 用户要求 LLM 分析但没有分析目标时，先问需要回答的问题或输出形态。

## 输出规则

- 生成完整可运行 Python 文件内容。
- 代码注释保持简洁。
- 不要把执行 playbook 的 MCP 操作混入 creator 工作流；执行交给 `asp-playbook-zh`。
- 不要编造不存在的内部服务。需要调用服务时，先搜索 backend 中已有 service/helper。

## 失败处理

- 如果无法确定 playbook 类型，只问一个聚焦问题。
- 如果缺少 case 上下文，生成代码时必须显式 `raise ValueError(...)`。
- 如果外部接口细节缺失，不要生成伪造 endpoint 或 token。
