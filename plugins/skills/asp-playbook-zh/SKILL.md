---
name: asp-playbook-zh
description: '操作 ASP playbook definition 和 playbook run 记录。适用于查看可运行的 playbook、对 case 执行 playbook，或查看已有 playbook run。'
argument-hint: 'list playbook definitions | run playbook <name> for <case_id> | list playbook runs [filters]'
compatibility: connect to asp mcp server
metadata:
  author: Funnywolf
  version: 0.1.1
  mcp-server: asp
  category: cyber security
  tags: [ playbook, automation, soar, investigation ]
  documentation: https://asp.viperrtp.com/
---

# ASP Playbook

当用户要在 ASP 中处理 playbook 自动化时，使用这个 skill。

## 适用场景

- 用户想知道当前有哪些 playbook definition 可以运行。
- 用户想对 case 执行 playbook。
- 用户想按目标 case 或 job status 查看 playbook run 记录。
- 用户想确认某个 case 是否已经执行过自动化。

## 运行规则

- 在语言和工作流层面，严格区分 playbook definition 与 playbook run record。
- `list_playbook_templates` 只用于查询可运行的 definition。
- `list_playbooks` 只用于查询运行记录。
- 只有当用户给出了可运行的 definition 名称和目标对象时，才使用 `execute_playbook`。
- 不要编造 playbook definition 名称。如果名称缺失，先列出或建议可用 definitions。
- 把 `user_input` 视为该次运行的自然语言补充说明，而不是通用聊天提示。

## 决策流程

1. 如果用户想知道哪些 playbook 可以运行，调用 `list_playbook_templates`。
2. 如果用户想确认某个 case 是否执行过自动化，调用 `list_playbooks(case_id=<case_id>, include_related=False, include_comments=False)`。
3. 如果用户要执行自动化，并已提供 definition 名称和目标 case，调用 `execute_playbook`。
4. 如果用户想执行自动化但不知道 definition 名称，先调用 `list_playbook_templates`。
5. 如果用户要看整体自动化历史，使用最窄有用过滤条件调用 `list_playbooks`。

## MCP 工具契约

- `list_playbook_templates()`
  - 返回后端 playbook definition registry 中可运行的模板。
- `execute_playbook(name, case_id, user_input=None)`
  - `name` 必须匹配可运行的 playbook template 名称。
  - `case_id` 是 `case_000001` 这类可读 ID。
  - `user_input` 是可选的本次运行补充说明。
  - 调用后会创建 pending playbook run 记录。
- `list_playbooks(playbook_id=None, job_status=None, case_id=None, include_related=False, limit=10, include_comments=False, comments_limit=20)`
  - `playbook_id` 是 `playbook_000001` 这类 run ID。
  - `job_status`：`Success`、`Failed`、`Pending`、`Running`。
  - `case_id` 用于过滤某个 case 关联的 runs。
  - `include_related=False` 返回紧凑 run 记录。只有审查具体某条 run 且需要关联 case 简要信息时才设为 `True`。
  - `include_comments=True` 会包含该 run 最近的 comments；`comments_limit` 默认 20，最大 50。
  - comment 附件只返回元数据和下载地址；需要下载文件时使用 `asp-file-zh` / `get_file`。
  - `limit` 会被限制在 1-100。

## SOP

### 列出可运行的 Playbook Definition

1. 调用 `list_playbook_templates`。
2. 解析返回的 JSON。
3. 只展示与用户目标对象或目标最相关的 definitions。
4. 明确说明这些是 definitions，而不是 run records。

### 运行一个 Playbook

1. 要求提供目标 case ID 和 playbook definition `name`。
2. 如果 definition 名称缺失或不确定，先调用 `list_playbook_templates`。
3. 只有在用户想提供额外指导时，才传 `user_input`。
4. 调用 `execute_playbook(case_id=<case_id>, name=<definition_name>, user_input=<optional>)`。
5. 确认已创建一条待执行的 playbook run 记录。

首选回复结构：

- `Target Case`：case ID
- `Playbook Definition`：选定的名称
- `Run Status`：创建时通常为 pending，除非平台返回其他状态
- `User Input`：仅在提供时展示
- `Next Useful Step`：通常是继续查询相关 playbook run

### 查看 Playbook Run

1. 提取支持的过滤字段：`playbook_id`、`job_status`、`case_id`、`limit`。
2. 当用户是从某个 case 视角提问时，优先使用 `case_id`。
3. 历史/列表视图使用 `include_related=False, include_comments=False`；只有具体某条 run 需要关联 case 上下文时才用 `include_related=True`，需要评论时才用 `include_comments=True`。
4. 调用 `list_playbooks`。
5. 解析返回的 JSON 字符串。
6. 用简短的 run 视图呈现。

首选回复结构：

| Run ID | Case ID | Job Status | Definition Name | Created |
|--------|---------|------------|-----------------|---------|

然后在需要时补一句简短解释。

## 澄清规则

- 只有在运行请求缺失目标 case ID 时，才询问。
- 只有在 definition 名称缺失或含糊时，才追问 playbook 名称。
- 如果用户给出的名字更像 run ID 而不是 definition，执行前先澄清。
- 如果用户说“check the run”但没有给 run ID，优先根据对象上下文用 `list_playbooks`，而不是猜某条具体 run。

## 输出规则

- 保持简洁。
- 不要混淆 definition、run、record 和 target object 这些概念。
- 如果只有少数 definitions 相关，不要把所有 playbook definition 全部倾倒出来。
- 优先使用运维语义：哪些能运行、哪些已经运行、哪些还在 pending、下一步该检查什么。

## 失败处理

- 如果 MCP 工具调用返回连接错误或超时，直接回复失败，提示用户检查 `ASP_MCP_URL`、`ASP_MCP_API_KEY`、ASGI `/api/mcp` 是否可访问，以及 API key 是否过期、用户是否被禁用。不要尝试重试或绕过。
- 如果没有匹配的 playbook definition，直接说明并给出最接近的相关选项。
- 如果目标对象没有任何 run 记录，直接说明。
- 如果执行前置条件缺失，只问一个聚焦问题，不要猜测。
- 如果用户的问题只能通过 run 记录回答，不要仅根据 definition 做回答。
