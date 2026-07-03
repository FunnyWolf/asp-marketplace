---
name: asp-playbook-creator
description: "创建或改进 ASP playbook。适用于编写 LLM 分析类或 SOAR 自动化处理类 playbook。"
argument-hint: "创建 playbook <目标>"
compatibility: requires asp-cli >= 0.1.0
disable-model-invocation: true
metadata:
  author: Funnywolf
  version: 1.0.0
  cli: asp
  category: cyber security
  tags: [playbook, automation, authoring, response]
  documentation: https://asp.viperrtp.com/
---

# ASP Playbook Creator

当用户想新建或修改 ASP playbook 脚本时，使用这个 skill。它只指导**编写 playbook definition**，不负责执行 playbook 或查询 playbook run；执行和查询交给 `asp-playbook`。

默认使用用户的语言回复；如果用户没有明确语言偏好，使用中文。

## 前置检查

如果 ASP CLI 尚未配置或认证状态不明确，先运行 `/asp-setup`。也可以直接执行：

```bash
asp doctor --output json
```

## 适用场景

- 用户要在 `backend/custom/playbooks/` 下新建自定义 playbook。
- 用户要改进已有 playbook definition。
- 用户要编写 LLM 智能分析、提示词研判、自动分诊类 playbook。
- 用户要编写调用内部服务或外部接口的 SOAR 自动化处理 playbook。

## 当前代码契约

### 文件位置和加载顺序

- 系统内置 playbook 位于 `backend/playbooks/`。
- 自定义 playbook 位于 `backend/custom/playbooks/`。
- 后端扫描顺序是内置目录后扫描 custom 目录；同名文件时 custom 文件覆盖内置文件。
- 推荐新增或定制 playbook 时写入 `backend/custom/playbooks/<playbook_file>.py`，不要修改内置 `backend/playbooks/`。
- 文件名建议使用 snake_case，且不要使用相对 import；loader 明确不支持相对 import。

### 类定义

每个 playbook 文件必须定义：

```python
class Playbook(BasePlaybook):
    NAME = "可读的 Playbook 名称"
    DESC = "说明该 playbook 的用途。"
    TAGS = ["Custom"]

    def run(self):
        ...
```

规则：

- 类名必须是 `Playbook`。
- 必须继承 `apps.agentic.runtime.base.BasePlaybook`。
- `NAME` 是可执行的 playbook template 名称，`asp playbook run <template_name> <case_id> --output json` 会精确匹配它。
- `DESC` 和 `TAGS` 会显示在 `asp playbook template list --output json`。
- `NAME` 应全局唯一；如果多个文件定义同名 `NAME`，查找结果会受文件扫描顺序影响，不要依赖这种行为。
- `run()` 不接收参数，返回字符串；worker 会把返回值写入 playbook run 的 `remark`。
- 缺少必要上下文时显式 `raise ValueError(...)`，不要静默返回成功。

### 运行上下文

`BasePlaybook.__init__` 会设置：

- `self.playbook_run`：当前 playbook run 记录。
- `self.case`：当前 run 关联的 Case；没有关联 case 时为 `None`。
- `self.user_input`：用户执行 playbook 时传入的补充说明。

worker 流程：

1. 认领 `Pending` run。
2. 标记为 `Running`，写入 `job_id`。
3. 执行 `Playbook(playbook_run=playbook_run).run()`。
4. 成功时标记 `Success`，并把 `str(result)` 写入 `remark`。
5. 失败时标记 `Failed`，并把异常类型和消息写入 `remark`。
6. 成功或失败都会发送 playbook completion 通知。

### Prompt 读取

最新代码已经在 `BasePlaybook` 中提供 prompt 辅助方法：

```python
self.read_prompt("System")
```

规则：

- prompt 默认从 `backend/custom/data/playbooks/<prompt_slug>/<PromptName>_<language>.md` 读取。
- `prompt_slug` 优先使用类属性 `PROMPT_SLUG`。
- 如果未设置 `PROMPT_SLUG`，并且 playbook 是从脚本文件加载的，则使用脚本文件名 stem。
- 因此自定义 LLM playbook 推荐显式设置 `PROMPT_SLUG`，并把提示词放到 `backend/custom/data/playbooks/<slug>/`。

示例：

```text
backend/custom/playbooks/case_summary.py
backend/custom/data/playbooks/case_summary/System_zh.md
backend/custom/data/playbooks/case_summary/System_en.md
```

## 决策流程

1. 先确认 playbook 类型：
   - LLM 智能分析类：自定义分析逻辑、提示词、研判流程，可能根据输出继续写回。
   - SOAR 自动化处理类：调用内部服务或外部接口，完成富化、通知、工单、封禁、隔离等动作。
2. 确认目标 case 数据需求：是否需要 alerts、artifacts、comments、comment attachments、enrichments。
3. 确认是否需要 `self.user_input` 影响逻辑或 prompt。
4. 确认输出写到哪里：只写 run `remark`，还是额外创建 enrichment/comment，或调用外部系统。
5. LLM 智能分析类要确认 `PROMPT_SLUG` 和提示词文件。
6. 读取相关后端示例和模型/服务后再生成代码。
7. 生成代码后提醒用户通过 `asp playbook template list --output json` 检查是否被发现；执行仍交给 `asp-playbook`。

## 参考文件

- `backend/apps/agentic/runtime/base.py`：`BasePlaybook`、`PROMPT_SLUG`、`read_prompt()`。
- `backend/apps/agentic/services/playbooks.py`：playbook discovery、custom overlay、pending run 创建和状态更新。
- `backend/apps/agentic/runtime/loader.py`：脚本加载规则，尤其是“不支持相对 import”。
- `backend/apps/agentic/runtime/monitor.py`：worker 如何执行 `run()`。
- `backend/playbooks/investigation.py`：内置 LLM case investigation 示例。
- `backend/playbooks/knowledge_extraction.py`：内置知识提取示例。
- `backend/playbooks/threat_intelligence_enrichment.py`：内置 artifact TI 富化示例。
- `backend/custom/playbooks/case_summary.py`：自定义 LLM summary 示例，使用 `PROMPT_SLUG` 和 `self.read_prompt()`。
- `backend/custom/playbooks/cmdb_enrichment.py`：自定义 SOAR/CMDB enrichment 示例。

## LLM 智能分析类模板

适用于用户希望编写自己的分析逻辑、提示词、研判流程，或根据 LLM 输出继续做自动化写回的场景。

推荐结构：

```python
import json

from django.db import transaction
from langchain_core.messages import HumanMessage, SystemMessage

from apps.agentic.analysis.profiles import serialize_case_for_investigation
from apps.agentic.runtime.base import BasePlaybook
from integrations.llm.llmapi import LLMAPI


class Playbook(BasePlaybook):
    NAME = "自定义 Case 摘要"
    DESC = "为关联 case 生成面向分析师的自定义摘要。"
    TAGS = ["Custom", "LLM", "Case"]
    PROMPT_SLUG = "custom_case_summary"

    def run(self):
        if self.case is None:
            raise ValueError("自定义 Case 摘要 playbook 需要关联 case。")

        payload = {
            "case": serialize_case_for_investigation(self.case),
            "user_input": self.user_input,
        }
        result = LLMAPI(temperature=0.0).get_model().invoke(
            [
                SystemMessage(content=self.read_prompt("System")),
                HumanMessage(content=json.dumps(payload, ensure_ascii=False)),
            ]
        )
        summary = _content_as_text(result.content).strip()
        if not summary:
            raise ValueError("LLM 返回了空摘要。")

        with transaction.atomic():
            case = type(self.case).objects.select_for_update().get(pk=self.case.pk)
            case.summary = summary
            case.save(update_fields=["summary", "updated_at"])

        return f"Case 摘要已更新：{summary[:160]}"


def _content_as_text(content):
    if isinstance(content, str):
        return content
    if isinstance(content, list):
        parts = []
        for item in content:
            if isinstance(item, dict):
                parts.append(str(item.get("text", item)))
            else:
                parts.append(str(item))
        return "\n".join(parts)
    return str(content)
```

生成 LLM playbook 时：

- 优先使用 `self.read_prompt("System")`，不要自己拼接 prompt 路径。
- 明确设置 `PROMPT_SLUG`，方便提示词目录稳定。
- 如果只需要现有 case investigation 能力，可直接参考内置 `Investigation`，不要重写复杂逻辑。
- 如果要写回 case 字段，使用 transaction 并锁定目标记录。
- 如果只需要简短执行摘要，直接从 `run()` 返回字符串即可。

## SOAR 自动化处理类模板

适用于调用内部服务或外部接口完成富化、通知、工单、封禁、隔离等动作。

```python
from django.db import transaction

from apps.agentic.runtime.base import BasePlaybook
from apps.comments.services import create_record_comment
from apps.enrichments.models import Enrichment, EnrichmentProvider, EnrichmentType


class Playbook(BasePlaybook):
    NAME = "自定义 Artifact 富化"
    DESC = "处理关联 case 下的 artifacts，并写入结构化 enrichment。"
    TAGS = ["Custom", "SOAR", "Enrichment"]

    def run(self):
        if self.case is None:
            raise ValueError("自定义 Artifact 富化 playbook 需要关联 case。")

        stats = {
            "alerts": 0,
            "artifacts": 0,
            "unique": 0,
            "processed": 0,
            "errors": 0,
        }
        artifacts = _collect_unique_artifacts(self.case)
        stats["unique"] = len(artifacts)
        stats["alerts"] = self.case.alerts.count()
        stats["artifacts"] = sum(alert.artifacts.count() for alert in self.case.alerts.all())

        for artifact in artifacts:
            if not artifact.value:
                continue
            try:
                output = {
                    "summary": f"已处理 {artifact.type} {artifact.value}",
                    "artifact_id": artifact.artifact_id,
                    "source": self.NAME,
                }
                _upsert_artifact_enrichment(artifact, output)
                stats["processed"] += 1
            except Exception:
                stats["errors"] += 1

        _add_case_comment(
            self.playbook_run,
            f"{self.NAME} 执行完成：processed={stats['processed']}, errors={stats['errors']}",
        )

        return (
            f"{self.NAME} 执行完成。"
            f"alerts={stats['alerts']}, artifacts={stats['artifacts']}, "
            f"unique={stats['unique']}, processed={stats['processed']}, errors={stats['errors']}"
        )


def _collect_unique_artifacts(case):
    artifacts = {}
    for alert in case.alerts.prefetch_related("artifacts"):
        for artifact in alert.artifacts.all():
            artifacts[artifact.id] = artifact
    return list(artifacts.values())


@transaction.atomic
def _upsert_artifact_enrichment(artifact, output):
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
    enrichment.name = "自动化结果"
    enrichment.value = artifact.value
    enrichment.desc = output.get("summary", "自动化执行完成。")
    enrichment.data = output
    enrichment.full_clean()
    enrichment.save()
    return enrichment


def _add_case_comment(playbook_run, body):
    author = playbook_run.user if playbook_run and playbook_run.user_id else None
    if author is None or playbook_run.case_id is None:
        return None
    return create_record_comment(
        author=author,
        content_object=playbook_run.case,
        body=body,
    )
```

SOAR playbook 生成规则：

- 遍历 artifacts 时先去重，避免同一 artifact 被多个 alert 重复处理。
- 外部服务调用细节不明确时，不要生成伪造接口地址或 token。
- 可重复执行的写回逻辑应使用 upsert 模式，避免重复 enrichment。
- 写 comment 前检查 `playbook_run.user`；没有用户时不要伪造作者。
- 对每个 artifact 的单点失败不要让整个 run 崩溃，除非用户要求 fail-fast。

## 关联数据获取

- Case：`self.case`
- Alerts：`self.case.alerts.all()` 或 `prefetch_related("artifacts", "enrichments")`
- Artifacts：`alert.artifacts.all()`
- Case enrichments：`self.case.enrichments.all()`
- Alert enrichments：`alert.enrichments.all()`
- Comments：用 `ContentType` + `Comment` 查询目标对象评论。
- Comment attachments：通过 `comment.attachments.all()` 获取附件，使用 `attachment.file.open("rb")` 读取文件内容。

读取评论附件示例：

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

附件不限制文件类型。生成 playbook 时，不要默认把附件当作文本；应根据文件名、大小和实际内容选择解析逻辑。

## 写回方式

- Enrichment：适合结构化结果、富化数据、外部系统返回。
- Comment：适合自然语言评论、执行说明、交接信息。
- Case 字段：只在用户明确要求且字段语义匹配时写回；写回时使用 transaction。
- Run remark：适合简短执行摘要，由 `run()` 返回字符串即可。

## SOP

### 步骤 1 — 确认 playbook 类型

询问用户要写哪类 playbook：

- LLM 智能分析类。
- SOAR 自动化处理类。

### 步骤 2 — 确认输入与目标

确认：

- 目标 case 数据需要哪些部分：alerts、artifacts、comments、comment attachments、enrichments。
- 是否需要 `self.user_input` 影响逻辑或 prompt。
- LLM 智能分析类的 `PROMPT_SLUG` 和 prompt 文件。
- 输出只写 `remark`，还是还要写 enrichment/comment/case 字段或调用外部系统。
- 是否有破坏性或批量操作；如果有，必须要求用户明确确认安全边界。

### 步骤 3 — 读取参考实现

生成代码前，至少查看：

- `backend/apps/agentic/runtime/base.py`
- `backend/apps/agentic/services/playbooks.py`
- `backend/apps/agentic/runtime/loader.py`
- 一个同类型 `backend/playbooks/*.py` 或 `backend/custom/playbooks/*.py` 示例

### 步骤 4 — 生成 playbook 文件

生成完整 `backend/custom/playbooks/<playbook_file>.py` 内容，必须包含：

- `class Playbook(BasePlaybook)`
- `NAME`、`DESC`、`TAGS`
- 需要 prompt 时包含 `PROMPT_SLUG`
- `def run(self):`
- 必要的 `self.case` 检查
- 简短、可读的返回 remark
- 不使用相对 import

### 步骤 5 — 说明注册与执行

生成后提醒：

- `NAME` 就是 `asp playbook run <template_name> <case_id> --output json` 使用的 template 名称。
- 如果 worker 已运行，可能需要重启或重新加载后才能发现新文件。
- 可通过 `asp playbook template list --output json` 检查新 playbook 是否被发现。
- 执行 playbook 不属于 creator 工作流；如用户要执行，转交 `asp-playbook`。

## 澄清规则

- 用户没说明 playbook 类型时，先问 LLM 分析还是 SOAR 自动化。
- 用户没说明目标 case 数据需求时，按最小需要读取，避免默认拉取全部上下文。
- 用户要求外部接口调用但没有接口地址、认证方式或参数时，先问缺失项。
- 用户要求 LLM 分析但没有分析目标时，先问需要回答的问题或输出形态。
- 用户要求写回 case 字段、comment 或 enrichment 时，确认写回内容和幂等策略。

## 输出规则

- 生成完整可运行 Python 文件内容。
- 代码注释保持简洁。
- 不要把执行 playbook 的 ASP CLI 操作混入 creator 工作流；执行交给 `asp-playbook`。
- 不要编造不存在的内部服务。需要调用服务时，先搜索后端已有 service/helper。
- 对于 custom playbook，默认写入 `backend/custom/playbooks/`。

## 失败处理

- 如果无法确定 playbook 类型，只问一个聚焦问题。
- 如果缺少 case 上下文，生成代码时必须显式 `raise ValueError(...)`。
- 如果外部接口细节缺失，不要生成伪造接口地址或 token。
- 如果用户要求使用相对 import，说明 loader 不支持相对 import，并改用绝对 import。
