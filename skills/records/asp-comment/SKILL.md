---
name: asp-comment
description: "管理 ASP 评论。适用于查看或添加 case、alert、artifact、enrichment、knowledge、playbook 上的自然语言评论。"
argument-hint: "list comments <target_id> | add comment <target_id>"
compatibility: requires asp-cli >= 0.1.0
metadata:
  author: Funnywolf
  version: 1.0.0
  cli: asp
  category: cyber security
  tags: [comment, note, collaboration, handoff]
  documentation: https://asp.viperrtp.com/
---

# ASP Comment

当用户要给 ASP 记录添加自然语言评论、分析师备注、交接说明、回复、@ 用户或附件评论时，使用这个 skill。

默认使用用户的语言回复；如果用户没有明确语言偏好，使用中文。

## 前置检查

如果 ASP CLI 尚未配置或认证状态不明确，先运行 `/asp-setup`。也可以直接执行：

```bash
asp doctor --output json
```

## 适用场景

- 用户要求添加 note、comment、分析观察、交接说明或调查备注。
- 用户要查看某个 ASP 记录下已有的 comments。
- 用户要回复已有评论或 @ 用户。
- 用户要把已上传文件附加到评论。
- 用户给出本地文件路径，希望先上传再附加到评论。

## 运行规则

- 只有当用户明确要求持久化自然语言评论、回复或交接说明时，才执行写入。
- 评论内容是给人看的叙述文本；结构化证据、风险评分、IOC 上下文或机器可读分析应使用 `asp-enrichment`。
- 不要编造 target ID、comment ID 或 `file_key`。目标不明确时，询问精确记录 ID。
- 如果 `file_key`、`parent_id` 或 mentions 无效，不要静默跳过。
- 如果用户提供本地文件路径而不是 `file_key`，先使用 `asp file upload` 上传，再把返回的 `file_key` 附加到评论。

## 命令契约

### 查看评论

```bash
asp comment list case_000001 --output json
```

支持分页：

```bash
asp comment list case_000001 --page-size 20 --output json
```

`target_id` 必须是以下记录 ID 之一：

- `case_...`
- `alert_...`
- `artifact_...`
- `enrichment_...`
- `knowledge_...`
- `playbook_...`

### 添加评论

```bash
asp comment add case_000001 --body "分析师交接备注" --output json
```

从文件读取评论正文：

```bash
asp comment add case_000001 --body-file ./note.md --output json
```

添加回复和 mentions：

```bash
asp comment add case_000001 --body "已复核" --parent-id 123 --mentions alice,bob --output json
```

上传本地文件并附加到评论：

```bash
asp file upload ./evidence.txt --output json
asp comment add case_000001 --body "附加证据文件" --file-key <file_key> --output json
```

`body` 可以为空；但空正文时必须提供至少一个 `--file-key`。

## 决策流程

1. 判断用户是要查看 comments，还是添加 comment。
2. 查看 comments 时，确认 `target_id`，然后执行 `asp comment list <target_id> --output json`。
3. 添加 comment 时，确认 `target_id`、正文、附件、回复目标和 mentions。
4. 如果用户给的是本地文件路径，先执行 `asp file upload <path> --output json`。
5. 如果用户给的是 `file_key`，直接使用 `--file-key`。
6. 如果内容是结构化证据或机器可读分析，转向 `asp-enrichment`。

## SOP

### 添加评论

1. 确认精确 `target_id`。
2. 提取 comment body、附件、回复目标和 @ 用户。
3. 尽量保留用户原意，只做必要格式整理。
4. 执行 `asp comment add`。
5. 确认 comment 已添加，并给出 comment ID。
6. 如果附带附件，列出文件名和 `file_key`。

首选回复结构：

- `Target`：目标 ID。
- `Comment`：创建后的 comment ID。
- `Attachments`：仅在有附件时列出文件名和 `file_key`。

### 查看评论

1. 确认 `target_id`。
2. 执行 `asp comment list <target_id> --output json`。
3. 只总结与用户问题相关的评论。
4. 附件只列文件名、类型、大小和 `file_key`；不要声称已经读取附件内容。

## 澄清规则

- 缺少或无法确定 `target_id` 时，询问。
- 既没有 comment body，也没有 `file_key` 或可上传文件时，询问要写入的评论内容或要附加的文件。
- 用户说“这个文件”但没有给出本地路径或 `file_key` 时，询问文件来源。
- 用户说“保存这个结果”且内容是结构化证据时，转向 enrichment 而不是 comment。

## 输出规则

- 保持简洁。
- 除非用户要求，不要重复很长的 comment body。
- 写入成功后明确给出 comment ID。
- 对附件只输出必要元数据，不输出大段内容。

## 失败处理

- 如果 ASP CLI 返回连接错误或超时，直接回复失败，提示用户运行 `asp doctor --output json`，检查 API URL、API key、网络连通性、用户状态和权限。不要尝试绕过认证。
- 如果目标记录不存在，直接说明。
- 如果 `file_key`、`parent_id` 或 mentions 无效，直接说明失败原因；不要静默跳过。
