---
name: asp-enrichment-zh
description: '把结构化数据保存为 enrichment，并附加到 case、alert 或 artifact。'
argument-hint: 'create enrichment <target_id> [fields]'
compatibility: connect to asp mcp server
metadata:
  author: Funnywolf
  version: 0.2.0
  mcp-server: asp
  category: cyber security
  tags: [ enrichment, analysis, context, investigation ]
  documentation: https://asp.viperrtp.com/
---

# ASP Enrichment

当数据需要以结构化上下文形式保存回 ASP 且挂载到对应 case、alert 或 artifact 时，使用这个 skill。

## 适用场景

- 用户想保存结构化分析、情报或调查结论。
- 用户想把上下文附加到 case、alert 或 artifact。
- 用户想持久化 SIEM 发现、威胁情报、资产上下文或分析师结论。

## 运行规则

- 把 enrichment 视为平台的结构化结果层，而不是普通评论字段。
- 当目标是把分析结果持久化到 `case`、`alert` 或 `artifact` 上时，使用这个 skill。
- 使用 `create_enrichment` 创建 enrichment 记录并自动附加到目标对象。
- enrichment payload 保持紧凑且可操作。
- 查看对象本身时优先使用对象对应的 skill；保存结果时再使用本 skill。
- 如果用户只是要添加自然语言评论，使用 `asp-comment-zh`。

## 决策流程

1. 如果用户想在目标对象上保存新的结构化结果，调用 `create_enrichment(target_id=..., ...)`。
2. 如果用户还处于对象探索阶段而不是保存结果，先使用对应对象 skill。

当你已经有明确的分析结论，例如 verdict、TTP 集合、风险评级或缓解建议，并且这些内容需要保存在目标对象上时，就切换到这个 skill。

## MCP 工具契约

- `create_enrichment(target_id, name="", type="Other", value="", uid="", desc="", data="")`
  - `target_id` 必须以 `case_`、`alert_` 或 `artifact_` 开头。
  - `name` 是简短 enrichment 标题。
  - `type` 应使用 ASP enrichment 类型，例如 `Threat Intelligence`、`Reputation`、`Geo Location`、`DNS`、`Vulnerability`、`Asset`、`CMDB`、`Identity`、`Behavior`、`Detection`、`Correlation`、`History`、`Remediation`、`Observation`、`External Ticket`、`Other`。
  - `value` 是被富化的主要值，例如 IOC、用户名、主机、case verdict 或 rule name。
  - `uid` 是可选的外部稳定标识，用于去重。
  - `desc` 是简短可读摘要。
  - `data` 必须是 JSON object 或包含 JSON object 的字符串；不要传数组或普通文本。
  - 后端会把 `provider` 固定为 `MCP`。

## SOP

### 创建 Enrichment

1. 要求提供 `target_id`（如 case_000001 / alert_000001 / artifact_000001）。
2. 把用户的分析整理成紧凑的结构化 enrichment payload。
3. 调用 `create_enrichment(target_id=<target_id>, name=..., type=..., value=..., uid=..., desc=..., data=...)`。
4. 确认创建后的 `enrichment_id` 及其已附加到目标对象。

首选回复结构：

- `Target ID`：目标 ID
- `Enrichment`：创建出的 enrichment ID

## 澄清规则

- 只有在缺失时才询问 `target_id`。
- 如果用户只说”把这个结果保存一下”，在上下文明确时推断最明显的目标对象，优先选择 Case。

## 输出规则

- 保持简洁。
- 除非用户明确要求，否则不要输出原始 JSON。
- 优先使用面向分析师的措辞，而不是存储层措辞。
- 明确说明保存了什么、附加到了哪里，以及它为什么有用。

## 失败处理

- 如果 MCP 工具调用返回连接错误或超时，直接回复失败，提示用户检查 `ASP_MCP_URL`、`ASP_MCP_API_KEY`、ASGI `/api/mcp` 是否可访问，以及 API key 是否过期、用户是否被禁用。不要尝试重试或绕过。
- 如果目标对象不存在，直接说明。
- 如果 enrichment payload 不完整，只问一个聚焦问题，不要猜测。
