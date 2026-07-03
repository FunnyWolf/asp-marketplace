---
name: asp-alert
description: "管理 ASP 告警。适用于列出、查看、审查告警，或围绕告警做分诊上下文判断。"
argument-hint: "list alerts | show alert <alert_id>"
compatibility: requires asp-cli >= 0.1.0
metadata:
  author: Funnywolf
  version: 1.0.0
  cli: asp
  category: cyber security
  tags: [alert, SOC, detection, triage]
  documentation: https://asp.viperrtp.com/
---

# ASP Alert

当用户要围绕 ASP 告警开展 SOC 分析工作时，使用这个 skill。Alert 是 ASP 中的二级数据，每个 Alert 挂载到一个 Case，并可关联一个或多个 Artifact。

默认使用用户的语言回复；如果用户没有明确语言偏好，使用中文。

## 前置检查

如果 ASP CLI 尚未配置或认证状态不明确，先运行 `/asp-setup`。也可以直接执行：

```bash
asp doctor --output json
```

## 适用场景

- 用户给出 alert ID，希望快速查看、审查或总结告警。
- 用户希望按状态、严重级别、置信度、case ID 或 correlation UID 查找告警。
- 用户想理解告警触发规则、source UID、rule ID、rule name、关联 case 或 artifacts。
- 用户想把分析结果保存到告警上；此时本 skill 负责判断并转交 `asp-enrichment`。
- 用户想在告警上添加自然语言评论；此时转交 `asp-comment`。

## 运行规则

- 回复要聚焦分诊价值，而不是原样回显 schema 字段。
- Alert 当前通过 CLI 主要提供查询能力；需要保存分析结果时使用 enrichment。
- 不要在缺少证据时把告警判定为 true positive 或 false positive。
- 如果用户问的是 case 级结论，转向 `asp-case` 或 `asp-case-investigation`。
- 如果用户问的是 artifact 级 pivot，转向 `asp-artifact` 或 `asp-artifact-investigation`。

## 命令契约

### 列出告警

```bash
asp alert list --output json
```

支持的常用过滤参数：

```bash
asp alert list --status "New" --severity "High" --confidence "Medium" --case-id case_000001 --correlation-uid "<uid>" --page-size 20 --output json
```

- `status`：`Unknown`、`New`、`In Progress`、`Suppressed`、`Resolved`、`Archived`、`Deleted`、`Other`。
- `severity`：`Unknown`、`Informational`、`Low`、`Medium`、`High`、`Critical`。
- `confidence`：`Unknown`、`Low`、`Medium`、`High`。
- `case_id` 用于查找挂载在指定 case 下的告警。
- `correlation_uid` 用于查找同一 correlation 下的告警。
- 需要关联 artifacts 时加 `--include-related`。
- 需要评论时额外执行 `asp comment list <alert_id> --output json`。

### 查看单条告警

```bash
asp alert show alert_000001 --include-related --output json
```

快速查看、不需要关联 artifacts 时：

```bash
asp alert show alert_000001 --no-include-related --output json
```

## 决策流程

1. 如果用户提供了具体 alert ID，或要求 open/show/review/summarize 某条告警，调用 `asp alert show <alert_id> --include-related --output json`。
2. 如果用户只需要快速查看基本信息，调用 `asp alert show <alert_id> --no-include-related --output json`。
3. 如果用户要浏览或对比多条告警，调用 `asp alert list`，只使用 CLI 明确支持的过滤条件。
4. 如果用户要求查看评论，额外调用 `asp comment list <alert_id> --output json`。
5. 如果用户要附加分析结果、情报或结构化上下文，转交 `asp-enrichment`。
6. 如果用户要添加自然语言评论，转交 `asp-comment`。

## SOP

### 审查单条告警

1. 获取告警详情和关联 artifacts。
2. 如果只需要快速查看，避免拉取不必要上下文。
3. 如果找不到告警，直接说明。
4. 只呈现对分诊有价值的字段。
5. 如果告警暴露出明确 artifact 且用户问题需要继续下钻，再建议使用 artifact、TI、CMDB 或 SIEM skill。

首选回复结构：

- `Alert`：alert ID、标题、严重级别、状态、置信度、correlation UID。
- `Timeline`：创建时间和更新时间。
- `Detection Context`：source UID、rule ID、rule name、关联 case ID。
- `Artifacts`：只列和当前问题相关的 artifacts。
- `Assessment`：简短分诊判断，明确区分事实与推断。

### 列出告警

1. 提取支持的过滤字段：`status`、`severity`、`confidence`、`case_id`、`correlation_uid`、`page_size`。
2. 将自然语言过滤条件规范化为 CLI 参数。
3. 调用 `asp alert list --output json` 并附加必要过滤条件。
4. 解析返回的 JSON。
5. 以紧凑对比视图呈现。

首选列表结构：

| Alert ID | Title | Severity | Status | Confidence | Case ID | Created | Rule Name |
| --- | --- | --- | --- | --- | --- | --- | --- |

## 澄清规则

- 单条告警操作缺少 `alert_id` 时，询问。
- 请求值无法清晰映射到 ASP 枚举时，询问。
- 用户要求更新告警字段时，说明当前 CLI 不暴露告警更新命令；可改为创建 enrichment 或添加 comment。
- 用户要求 case 级判断时，不要只停留在 alert 视角，应建议或转向 case 级 skill。

## 输出规则

- 保持简洁，优先使用分诊语义。
- 除非用户明确要求，否则不要输出原始 JSON。
- 清楚指出阻塞项：告警不存在、不支持的过滤条件、无效枚举值、认证失败或权限不足。
- 如果一次分析用了多个 CLI 调用，把结果合并成一段连贯的告警叙事，不要逐个调用回显。

## 失败处理

- 如果 ASP CLI 返回连接错误或超时，直接回复失败，提示用户运行 `asp doctor --output json`，检查 API URL、API key、网络连通性、用户状态和权限。不要尝试绕过认证。
- 如果告警不存在，直接说明。
- 如果过滤无结果，直接说明并建议最有用的收敛方式。
