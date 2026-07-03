---
name: asp-knowledge
description: "检索、查看或更新 ASP knowledge 记录。适用于内部经验、SOP、历史模式和分析师知识复用。"
argument-hint: "search knowledge [keyword] | show knowledge <knowledge_id> | update knowledge <knowledge_id>"
compatibility: requires asp-cli >= 0.1.0
metadata:
  author: Funnywolf
  version: 1.0.0
  cli: asp
  category: cyber security
  tags: [knowledge, SOP, reuse, investigation]
  documentation: https://asp.viperrtp.com/
---

# ASP Knowledge

当用户要在 ASP 中检索或维护内部知识库时，使用这个 skill。Knowledge 用来承载可复用经验、SOP、历史模式、误报处理经验和调查建议。

默认使用用户的语言回复；如果用户没有明确语言偏好，使用中文。

## 前置检查

如果 ASP CLI 尚未配置或认证状态不明确，先运行 `/asp-setup`。也可以直接执行：

```bash
asp doctor --output json
```

## 设计思路

Knowledge 是数据库中的知识记录，核心字段包括：

- `title`：知识标题。
- `body`：知识主体内容，支持 Markdown。
- `tags`：标签，用于筛选和分类。
- `source`：知识来源，例如手动创建或从 case 中提取。
- `expires_at`：过期时间，留空表示长期有效。

`asp knowledge search` 会匹配 title、body、tags。不要声称 CLI 会自动过滤过期记录，除非返回数据明确体现这一点。

## 适用场景

- 用户想通过关键词查找内部知识、SOP、历史案例经验或处理建议。
- 用户想查看某条 knowledge 的完整内容。
- 用户想更新知识条目的标题、正文、标签或过期时间。
- 用户想把当前调查结论沉淀为结构化对象；这种情况通常应转向 `asp-enrichment`，除非用户明确要维护 knowledge。

## 运行规则

- Knowledge 用于支持当前判断，不替代当前证据。
- 如果某条 knowledge 改变了建议，说明它如何影响判断。
- 不要更新 knowledge，除非用户明确要求编辑已有记录。
- 如果用户只是要给 case 保存本次调查发现，优先使用 `asp-enrichment` 或 `asp-comment`。
- 除非用户明确要求，不要输出完整 knowledge body。

## 命令契约

### 搜索 knowledge

```bash
asp knowledge search "phishing" --output json
```

支持常用过滤：

```bash
asp knowledge search "aws" --source Manual --case-id case_000001 --tags "cloud,iam" --page-size 20 --output json
```

- `keyword` 可以是主题、问题描述、症状、IOC 类型或自然语言查询。
- 多个 tag 使用逗号分隔。
- `case_id` 用于查找关联到指定 case 的知识。
- 需要评论时额外执行 `asp comment list <knowledge_id> --output json`。

### 查看单条 knowledge

```bash
asp knowledge show knowledge_000001 --output json
```

### 更新 knowledge

仅当用户明确要求更新时执行：

```bash
asp knowledge update knowledge_000001 --title "更新后的标题" --body-file ./knowledge.md --expires-at 2026-06-23T12:00:00Z --tags "cloud,iam" --output json
```

- `knowledge_id` 是 `knowledge_000001` 这类可读 ID。
- `body` 支持 Markdown；用户提供结构化内容时，应保留标题、列表、表格、代码块和链接。
- `expires_at` 必须是带时区的 ISO 8601，例如 `2026-06-23T12:00:00Z` 或 `2026-06-23T20:00:00+08:00`。
- `tags` 使用逗号分隔。

## 决策流程

1. 如果用户要搜索相关经验或 SOP，调用 `asp knowledge search`。
2. 如果用户要查看具体记录，调用 `asp knowledge show <knowledge_id> --output json`。
3. 如果用户要修改已知记录，调用 `asp knowledge update`，并且只携带用户明确要求修改的字段。
4. 如果用户要保存当前调查发现，先判断更适合 comment、enrichment 还是 knowledge；不要默认写入 knowledge。

## SOP

### 搜索 knowledge

1. 将用户问题、关键词或场景描述整理为搜索关键词。
2. 调用 `asp knowledge search <keyword> --output json`。
3. 返回最相关的少量结果，优先展示标题、标签、来源和简短说明。
4. 只有用户要求或需要判断依据时，才展开完整 body。

首选列表结构：

| Knowledge ID | Title | Source | Tags | Why Relevant |
| --- | --- | --- | --- | --- |

### 查看 knowledge

1. 确认 `knowledge_id`。
2. 调用 `asp knowledge show <knowledge_id> --output json`。
3. 如果内容很长，只摘要与用户问题相关的部分。
4. 如果 knowledge 已过期或看起来不适用，明确说明。

### 更新 knowledge

1. 确认 `knowledge_id`。
2. 只提取用户明确要求修改的字段：`title`、`body`、`expires_at`、`tags`。
3. 如果正文较长，优先建议使用 `--body-file`。
4. 仅带变更字段调用 `asp knowledge update`。
5. 只确认实际变更的字段。

首选回复结构：

- `Updated knowledge`：knowledge ID。
- `Changed fields`：实际更新字段。

## 澄清规则

- 用户要更新特定记录但未提供 `knowledge_id` 时，询问。
- 搜索意图太宽泛时，询问一个更具体的主题、技术、告警类型、IOC 类型或处置场景。
- 用户说“保存这个结果”但没有说明保存为 knowledge、comment 还是 enrichment 时，先判断并说明推荐方式。

## 输出规则

- 保持简洁。
- 除非用户明确要求，否则不要输出完整 knowledge body。
- 匹配记录很多时，展示最有价值的子集，并说明可用哪些关键词或标签继续收敛。
- 明确区分 knowledge 中的历史经验和当前 case 的实时证据。

## 失败处理

- 如果 ASP CLI 返回连接错误或超时，直接回复失败，提示用户运行 `asp doctor --output json`，检查 API URL、API key、网络连通性、用户状态和权限。不要尝试绕过认证。
- 如果搜索没有结果，直接说明，并建议换一组关键词、标签或 case ID。
- 如果要更新的记录不存在，直接说明。
