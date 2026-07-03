---
name: asp-artifact
description: "查询和分析 ASP artifact。适用于围绕 IP、用户、主机、域名、URL、hash 等对象做调查 pivot。"
argument-hint: "list artifacts | show artifact <artifact_id>"
compatibility: requires asp-cli >= 0.1.0
metadata:
  author: Funnywolf
  version: 1.0.0
  cli: asp
  category: cyber security
  tags: [artifact, IOC, pivot, investigation]
  documentation: https://asp.viperrtp.com/
---

# ASP Artifact

当用户要围绕 artifact 做对象级调查时，使用这个 skill。Artifact 是 ASP 中的最小调查对象，可以代表 IP、用户名、主机名、域名、URL、hash、进程、资源等实体。

默认使用用户的语言回复；如果用户没有明确语言偏好，使用中文。

## 前置检查

如果 ASP CLI 尚未配置或认证状态不明确，先运行 `/asp-setup`。也可以直接执行：

```bash
asp doctor --output json
```

## 适用场景

- 用户想按 artifact ID、value、type 或 role 查找 artifact。
- 用户给出 IOC 或对象值，希望确认 ASP 中是否已有对应 artifact。
- 用户想查看 artifact 关联过哪些 alert 或 case。
- 用户想从 artifact 出发继续做 TI、CMDB、SIEM 或 case pivot。
- 用户想把分析结果保存到 artifact 上；此时转交 `asp-enrichment`。

## 运行规则

- 把 artifact 视为调查 pivot，不要把 artifact 存在本身当作最终结论。
- 只查询和分析已有 artifact；如果用户要新建对象，当前 CLI 不提供直接创建 artifact 的命令。
- 如果用户需要资产、身份、负责人或业务上下文，使用 `asp-cmdb`。
- 如果用户需要 IOC 声誉或外部威胁情报，使用 `asp-threat-intelligence`。
- 如果用户需要事件证据、时间线、流行度或影响范围，使用 `asp-siem-search`。
- 如果用户需要保存结构化结论，使用 `asp-enrichment`。

## 命令契约

### 列出 artifact

```bash
asp artifact list --output json
```

支持的常用过滤参数：

```bash
asp artifact list --type "IP Address" --role Actor --value 8.8.8.8 --page-size 20 --output json
```

- `type` 可使用 ASP artifact 类型值，例如 `IP Address`、`Hostname`、`User Name`、`Email Address`、`URL String`、`Hash`、`File Name`、`Process Name`、`Resource UID`、`Port`、`Subnet`、`Command Line`、`CVE`、`Account`、`Resource`、`File Path`、`Device`、`Registry Path`、`Other` 等平台值。
- `role`：`Unknown`、`Target`、`Actor`、`Affected`、`Related`、`Other`。
- `value` 是 artifact 值的精确匹配。
- 多个 `type` 或 `role` 值优先使用 CLI 支持的逗号分隔方式。
- 需要关联 alert 时加 `--include-related`。
- 需要评论时额外执行 `asp comment list <artifact_id> --output json`。
- 当前 CLI 没有 owner 过滤条件。

### 查看单个 artifact

```bash
asp artifact show artifact_000001 --include-related --output json
```

快速查看、不需要关联 alert 时：

```bash
asp artifact show artifact_000001 --no-include-related --output json
```

## 决策流程

1. 如果用户提供 artifact ID，调用 `asp artifact show <artifact_id> --include-related --output json`。
2. 如果用户提供的是对象值或 IOC，先调用 `asp artifact list --value <value> --output json`。
3. 如果用户要浏览或对比 artifact，调用 `asp artifact list`，只使用 CLI 明确支持的过滤条件。
4. 如果用户要求查看评论，额外调用 `asp comment list <artifact_id> --output json`。
5. 如果用户要保存分析、情报或结构化上下文，转交 `asp-enrichment`。
6. 如果用户正从 artifact 出发调查，只建议下一个最有价值的 pivot，不要默认展开全量 hunt。

## SOP

### 查询 artifact

1. 从请求中提取最窄且最有用的过滤条件：`artifact_id`、`value`、`type`、`role`。
2. 列表查询默认不加 `--include-related`。
3. 审查具体 artifact 且需要关联 alert 时，使用 `--include-related`。
4. 解析返回的 JSON。
5. 如果用户下一步可能需要复用该对象，显式展示 `artifact_id`。

首选列表结构：

| Artifact ID | Value | Type | Role | Related Context |
| --- | --- | --- | --- | --- |

### 审查 artifact

1. 获取 artifact 详情。
2. 只在相关时展示关联 alert 或 case。
3. 判断它更适合做 TI、CMDB 还是 SIEM pivot。
4. 如果 artifact 不存在，直接说明。

首选回复结构：

- `Artifact`：artifact ID、value、type、role、name。
- `Observed In`：相关 alert 或 case；只列关键项。
- `Pivot Value`：为什么这个对象值得继续查，或为什么当前不值得继续查。
- `Next Step`：一个最有价值的下一步，例如 TI、CMDB、SIEM 或 case 查看。

## 澄清规则

- 用户要 enrich 现有 artifact 但没有提供 artifact ID 时，询问。
- 用户只给了模糊描述而没有对象值、type 或 artifact ID 时，询问最窄可查值。
- 用户要求“查这个对象是否恶意”时，说明需要 TI、SIEM 或 case 上下文，不能仅凭 artifact 存在判断。

## 输出规则

- 保持简洁，优先使用 pivot 语义，而不是存储语义。
- 除非用户明确要求，否则不要输出原始 JSON。
- 匹配很多时，展示最有价值的一小部分，并说明整体模式或建议收敛条件。
- 明确指出阻塞项：artifact 不存在、过滤条件不支持、认证失败或权限不足。

## 失败处理

- 如果 ASP CLI 返回连接错误或超时，直接回复失败，提示用户运行 `asp doctor --output json`，检查 API URL、API key、网络连通性、用户状态和权限。不要尝试绕过认证。
- 如果没有匹配的 artifact，直接说明，并建议最有用的收敛方式。
- 如果目标 artifact 不存在，直接说明。
