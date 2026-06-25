---
name: asp-cmdb-zh
description: '通过 CMDB provider 查询 ASP artifact 的资产、身份、邮箱、资源、负责人和业务上下文。'
argument-hint: 'lookup cmdb <artifact_type> <artifact_value> [provider]'
compatibility: connect to asp mcp server
metadata:
  author: Funnywolf
  version: 0.1.0
  mcp-server: asp
  category: cyber security
  tags: [ cmdb, asset, identity, context, investigation ]
  documentation: https://asp.viperrtp.com/
---

# ASP CMDB Context

当用户需要为可观测对象查询业务、资产、身份、邮箱、资源、负责人或关联上下文时，使用这个 skill。

## 适用场景

- 用户想知道某个 IP、主机名、账号、邮箱、资源、子网、端口或设备属于谁。
- 用户在分诊前需要 owner、业务系统、资产重要性或身份上下文。
- Artifact 调查除了 SIEM 证据或威胁情报外，还需要内部环境上下文。

## 运行规则

- CMDB 用于解释内部归属和业务语义，不要把它当成威胁情报。
- 如果 artifact 已存在于 ASP，优先使用 artifact 记录中的 type 和 value。
- 如果需要把有用 CMDB 上下文保存到 case、alert 或 artifact，使用 `asp-enrichment-zh`。

## MCP 工具契约

- `cmdb_lookup(artifact_type, artifact_value, provider=None)`
  - `artifact_type` 应使用 ASP artifact 类型，例如 `IP Address`、`Hostname`、`User Name`、`Email Address`、`Resource UID`、`Port`、`Subnet`、`Account`、`Resource`、`Device`、`Other`。
  - `artifact_value` 是要查询的精确值。
  - `provider` 可选。省略时查询所有已配置 CMDB provider；如果提供了未知名称，后端会报错。
  - 返回 `artifact_type`、`artifact_value`、各 provider 的 `results` 和 `errors`。
  - 每个 provider result 可能包含 `asset`、`identity`、`mailbox`、`resource`、`port`、`subnet`、`business`、`owner`、`related`、`raw`、`error`。

## SOP

1. 从用户请求或当前 artifact 上下文提取 `artifact_type` 和 `artifact_value`。
2. 如果类型有歧义，选择最可能的 ASP artifact type，并说明该假设。
3. 调用 `cmdb_lookup`。
4. 只总结会影响调查判断的上下文。

首选回复结构：

**CMDB Context**
- Observable：`<artifact_type> <artifact_value>`
- Provider
- Owner / 业务系统 / 重要性（如有）
- 资产或身份详情（如有）
- 有价值的关联对象

**Assessment**
- 用一句话说明该上下文如何影响分诊或下一步 pivot。

## 澄清规则

- 只有缺少 `artifact_value` 时才询问。
- 只有无法从值或当前 artifact 合理推断时，才询问 `artifact_type`。
- 不要询问 provider，除非用户明确要求选择 provider。

## 输出规则

- 保持简洁。
- 除非用户要求，否则不要倾倒原始 JSON。
- 区分 CMDB 事实和调查判断。

## 失败处理

- 如果 MCP 工具调用返回连接错误或超时，直接回复失败，提示用户检查 `ASP_MCP_URL`、`ASP_MCP_API_KEY`、ASGI `/api/mcp` 是否可访问，以及 API key 是否过期、用户是否被禁用。不要尝试重试或绕过。
- 如果 provider 返回错误，报告错误并建议检查 provider 配置。
- 如果没有 provider 支持该 artifact type，直接说明。
