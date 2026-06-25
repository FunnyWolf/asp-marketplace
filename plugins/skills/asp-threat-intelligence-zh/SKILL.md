---
name: asp-threat-intelligence-zh
description: '查询 IOC（IP、哈希、URL、域名）的威胁情报。当用户要检查某个指标是否恶意、评估风险等级或收集威胁上下文时使用。'
argument-hint: 'query ti <indicator> | query ti <indicator> from <provider>'
compatibility: connect to asp mcp server
metadata:
  author: Funnywolf
  version: 0.1.0
  mcp-server: asp
  category: cyber security
  tags: [ threat-intelligence, IOC, enrichment, investigation ]
  documentation: https://asp.viperrtp.com/
---

# ASP 威胁情报

当用户需要查询某个指标的威胁情报时，使用此 skill。支持 IP 地址、文件哈希、URL 和域名。

## 适用场景

- 用户要检查某个 IP、哈希、URL 或域名是否恶意。
- 用户要获取某个指标的风险评估或信誉数据。
- 用户要收集威胁上下文（标签、攻击技术、恶意软件家族）。
- 用户要将外部威胁情报附加到 artifact 或调查中。

## 运行规则

- 使用 `ti_query` 查询一个或所有已注册的威胁情报提供商。
- 省略 `provider` 参数时，工具会查询所有提供商并聚合结果。
- 如需指定提供商，传入 provider 名称（如 `"AlienVaultOTX"`）。
- 当需要将结果保存到现有 artifact 时，结合 `asp-enrichment-zh` skill 使用。

## 补充信息

- Provider 名称由部署环境决定。当前没有 MCP list-provider 工具；如果 provider 名称未知，省略 `provider` 或报告后端错误。
- 返回的 `aggregated_risk_level` 是所有提供商中最高的风险等级：high > medium > low。

## 决策流程

1. 用户给出指标（IP、哈希、URL、域名），直接调用 `ti_query`。
2. 用户指定了提供商名称，传入 `provider` 参数。
3. 用户要保存结果，使用 `asp-enrichment-zh` 持久化为 enrichment。

## MCP 工具契约

- `ti_query(indicator, artifact_type="Unknown", provider=None)`
  - `indicator` 是要查询的 IOC 值。
  - `artifact_type` 在已知时应传 ASP artifact 类型，例如 `IP Address`、`Hostname`、`URL String`、`Hash`、`Email Address` 或 `Unknown`。
  - `provider` 可选。省略时查询所有已配置 provider；如果提供了未知名称，后端会报错。
  - 返回 `indicator`、`indicator_type`、各 provider 的 `results`、`aggregated_risk_level` 和 `errors`。

## SOP

### 查询威胁情报

1. 从用户请求中提取指标。
2. 判断用户需要指定提供商还是查询所有提供商。
3. 调用 `ti_query(indicator=<值>, artifact_type=<类型或 "Unknown">, provider=<名称或 None>)`。
4. 解析响应并展示结果。

首选回复结构：

**指标概览**
- 指标值和检测到的类型
- 查询的提供商
- 聚合风险等级

**威胁发现**
- 各提供商的风险等级
- 信誉分数
- 标签、攻击技术、恶意软件家族、对手（如有）
- 网络上下文（ASN、国家等，仅 IP 指标）

**评估**
- 一行判定：恶意 / 可疑 / 安全 / 未知
- 如有必要，建议后续步骤

## 澄清规则

- 指标缺失或有歧义时，询问具体值。
- 用户说"查这个 IP"但给的是哈希，确认指标类型。
- 不要询问使用哪个提供商，除非用户明确要求选择。

## 输出规则

- 保持简洁。以风险等级和判定开头。
- 除非用户要求，否则不输出原始 JSON。
- 只突出显示关键字段：risk_level、tags、attack_techniques、malware_families。
- 如果指标是安全的，简要说明即可，不要过度解释阴性结果。
- 多个提供商都有结果时，总结共识并标注分歧。

## 失败处理

- 如果 MCP 工具调用返回连接错误或超时，直接回复失败，提示用户检查 `ASP_MCP_URL`、`ASP_MCP_API_KEY`、ASGI `/api/mcp` 是否可访问，以及 API key 是否过期、用户是否被禁用。不要尝试重试或绕过。
- 提供商返回错误时，报告错误并建议检查 API 配置。
- 指标格式无效时，说明该类型的期望格式。
- 没有可用的提供商时，直接报告。
