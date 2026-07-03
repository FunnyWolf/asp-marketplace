---
name: asp-playbook
description: "管理 ASP playbook 自动化。适用于列出模板、查看运行记录，或在明确授权后执行 playbook。"
argument-hint: "list templates | list runs | show run <playbook_id> | run <template> <case_id>"
compatibility: requires asp-cli >= 0.1.0
metadata:
  author: Funnywolf
  version: 1.0.0
  cli: asp
  category: cyber security
  tags: [playbook, automation, response]
  documentation: https://asp.viperrtp.com/
---

# ASP Playbook

当用户要在 ASP 中查看或执行 playbook 自动化时，使用这个 skill。

默认使用用户的语言回复；如果用户没有明确语言偏好，使用中文。

## 前置检查

如果 ASP CLI 尚未配置或认证状态不明确，先运行 `/asp-setup`。也可以直接执行：

```bash
asp doctor --output json
```

## 适用场景

- 用户想知道当前有哪些 playbook template 可以运行。
- 用户想对某个 case 执行 playbook。
- 用户想按目标 case 或 job status 查看 playbook run 记录。
- 用户想确认某个 case 是否已经执行过自动化。
- 用户想查看某次 playbook run 的状态、输入或结果。

## 运行规则

- 严格区分 playbook template 与 playbook run record。
- `asp playbook template list` 只用于查询可运行的 template。
- `asp playbook list` 和 `asp playbook show` 只用于查询运行记录。
- 只有用户明确要求执行自动化，并提供可运行 template 名称和目标 case 时，才使用 `asp playbook run`。
- 不要把“建议运行 playbook”直接变成执行动作。
- 不要编造 playbook template 名称；名称缺失或不确定时，先列出可用 templates。
- `user_input` 是该次运行的自然语言补充说明，不是通用聊天提示。

## 命令契约

### 列出可运行 template

```bash
asp playbook template list --output json
```

### 执行 playbook

仅当用户明确要求执行时运行：

```bash
asp playbook run template_name case_000001 --user-input "仅处理受影响用户" --output json
```

也可以从文件读取输入：

```bash
asp playbook run template_name case_000001 --user-input-file ./input.txt --output json
```

- `template_name` 必须匹配可运行的 playbook template 名称。
- `case_id` 是 `case_000001` 这类可读 ID。
- 调用后会创建 pending playbook run 记录，具体执行由后端 worker 处理。

### 列出 run 记录

```bash
asp playbook list --case-id case_000001 --job-status Pending --page-size 20 --output json
```

常见 `job_status`：`Success`、`Failed`、`Pending`、`Running`。

### 查看单条 run

```bash
asp playbook show playbook_000001 --include-related --output json
```

需要评论时额外执行：

```bash
asp comment list playbook_000001 --output json
```

## 决策流程

1. 用户想知道有哪些自动化可用时，调用 `asp playbook template list --output json`。
2. 用户想确认某个 case 是否执行过自动化时，调用 `asp playbook list --case-id <case_id> --output json`。
3. 用户要执行自动化且已提供 template 名称和目标 case 时，调用 `asp playbook run`。
4. 用户要执行自动化但不知道 template 名称时，先列出 templates。
5. 用户要看自动化历史时，使用最窄有用过滤条件调用 `asp playbook list` 或 `asp playbook show`。

## SOP

### 列出可运行 template

1. 调用 `asp playbook template list --output json`。
2. 解析返回的 JSON。
3. 只展示与用户目标 case 或响应目标最相关的 templates。
4. 明确说明这些是 templates，不是 run records。

### 执行 playbook

1. 确认目标 case ID。
2. 确认可运行的 playbook template 名称。
3. 如果 template 名称缺失或不确定，先调用 `asp playbook template list`。
4. 只有在用户想提供额外指导时，才传 `--user-input` 或 `--user-input-file`。
5. 执行 `asp playbook run <template_name> <case_id> --output json`。
6. 确认已创建 playbook run 记录，并说明初始状态。

首选回复结构：

- `Target Case`：case ID。
- `Playbook Template`：选定 template 名称。
- `Run`：创建出的 playbook run ID。
- `Run Status`：创建时状态。
- `Next Step`：通常是稍后查看 run 状态。

### 查看 playbook run

1. 提取支持的过滤字段：`playbook_id`、`job_status`、`case_id`、`page_size`。
2. 从 case 视角提问时，优先使用 `--case-id`。
3. 列表视图默认不加 `--include-related`。
4. 审查具体 run 且需要 case 上下文时，使用 `asp playbook show <playbook_id> --include-related --output json`。
5. 需要评论时额外执行 `asp comment list <playbook_id> --output json`。
6. 用简短 run 视图呈现。

首选列表结构：

| Run ID | Case ID | Job Status | Template | Created |
| --- | --- | --- | --- | --- |

## 澄清规则

- 执行请求缺少目标 case ID 时，询问。
- template 名称缺失或含糊时，询问或先列出 templates。
- 用户给出的名字更像 run ID 而不是 template 时，执行前先澄清。
- 用户说“check the run”但没有给 run ID 时，优先根据 case 上下文查询 run 列表，不要猜某条具体 run。

## 输出规则

- 保持简洁。
- 不要混淆 template、run record 和 target case。
- 如果只有少数 templates 相关，不要倾倒全部 template。
- 优先使用运维语义：哪些能运行、哪些已经运行、哪些 pending、下一步该检查什么。

## 失败处理

- 如果 ASP CLI 返回连接错误或超时，直接回复失败，提示用户运行 `asp doctor --output json`，检查 API URL、API key、网络连通性、用户状态和权限。不要尝试绕过认证。
- 如果没有匹配的 playbook template，直接说明并给出最接近的相关选项。
- 如果目标 case 没有任何 run 记录，直接说明。
- 如果执行前置条件缺失，只问一个聚焦问题，不要猜测。
- 如果用户的问题只能通过 run 记录回答，不要仅根据 template 做回答。
