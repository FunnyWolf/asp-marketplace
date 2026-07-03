---
name: asp-threat-intelligence
description: "查询 ASP 威胁情报。适用于获取 IOC 声誉、风险等级、标签、攻击技术和恶意上下文。"
argument-hint: "query <indicator> [artifact type]"
compatibility: requires asp-cli >= 0.1.0
metadata:
  author: Funnywolf
  version: 1.0.0
  cli: asp
  category: cyber security
  tags: [threat-intelligence, IOC, reputation, risk]
  documentation: https://asp.viperrtp.com/
---

# ASP Threat Intelligence

当用户需要查询某个指标的威胁情报时，使用此 skill。常见指标包括 IP 地址、文件哈希、URL、域名、邮箱和主机名。

默认使用用户的语言回复；如果用户没有明确语言偏好，使用中文。

## 前置检查

如果 ASP CLI 尚未配置或认证状态不明确，先运行 `/asp-setup`。也可以直接执行：

```bash
asp doctor --output json
```

## 适用场景

- 用户要检查某个 IP、hash、URL、域名或邮箱是否恶意。
- 用户要获取某个指标的风险评估、信誉数据或 provider 信号。
- 用户要收集威胁上下文，例如标签、攻击技术、恶意软件家族或攻击者信息。
- 用户要将外部威胁情报附加到 artifact、alert 或 case。

## 运行规则

- 威胁情报是辅助上下文，不是最终 verdict。
- 省略 provider 时，CLI 会查询所有已配置提供商并聚合结果。
- Provider 名称由部署环境决定；当前没有 provider 列表命令。provider 名称未知时，优先不传 provider。
- 区分“provider 报错”和“查询结果为低风险/无命中”。
- 需要保存结果时，必须在用户明确要求持久化后转交 `asp-enrichment`。
- 不要用威胁情报替代 SIEM 证据；需要验证环境内是否出现过该 IOC 时，使用 `asp-siem-search`。

## 命令契约

查询所有 provider：

```bash
asp ti query 8.8.8.8 --artifact-type "IP Address" --output json
```

指定 provider：

```bash
asp ti query example.com --artifact-type "Domain" --provider AlienVaultOTX --output json
```

参数说明：

- `indicator` 是要查询的 IOC 值。
- `artifact_type` 在已知时应传 ASP artifact 类型，例如 `IP Address`、`Hostname`、`URL String`、`Hash`、`Email Address` 或 `Unknown`。
- `provider` 可选。省略时查询所有已配置 provider；未知 provider 会由后端报错。
- 返回内容通常包括 `indicator`、`indicator_type`、各 provider 的 `results`、聚合风险等级和 `errors`。

## 决策流程

1. 从用户请求中提取 indicator。
2. 判断 artifact type；如果无法可靠判断，使用 `Unknown` 或询问。
3. 如果用户指定 provider，则传入 `--provider`；否则查询所有 provider。
4. 调用 `asp ti query`。
5. 解析响应，区分风险信号、provider 错误和无结果。
6. 如果用户要保存结果，转交 `asp-enrichment`。

## SOP

### 查询威胁情报

1. 提取指标值。
2. 判断指标类型：IP、domain、URL、hash、email、hostname 或 unknown。
3. 决定是否指定 provider。
4. 执行 `asp ti query <indicator> --artifact-type <type> --output json`。
5. 用风险等级和关键 provider 信号开头。
6. 只展开和当前调查目标相关的标签、技术、恶意软件家族或网络上下文。

首选回复结构：

**指标概览**

- `Indicator`：指标值。
- `Type`：识别或指定的类型。
- `Providers`：查询到的 provider。
- `Aggregated Risk`：聚合风险等级。

**威胁发现**

- 各 provider 风险等级或信誉分。
- 标签、攻击技术、恶意软件家族、攻击者信息。
- IP 指标可补充 ASN、国家或网络上下文。

**评估**

- 一行判断：恶意、可疑、安全、未知或数据不足。
- 下一步建议：例如 SIEM 验证、CMDB 归属查询、保存 enrichment。

## 澄清规则

- 指标缺失或有歧义时，询问具体值。
- 用户说“查这个 IP”但给出的值明显不是 IP 时，确认指标类型。
- 不要询问使用哪个 provider，除非用户明确要求选择 provider 或 provider 结果冲突。
- 用户要求“判断是否影响我们环境”时，说明还需要 SIEM 或 CMDB 上下文。

## 输出规则

- 保持简洁，以风险等级和判断开头。
- 除非用户要求，否则不输出原始 JSON。
- 只突出关键字段：risk level、tags、attack techniques、malware families、provider errors。
- 如果指标风险较低或无命中，简要说明即可，不要过度解释阴性结果。
- 多个 provider 有结果时，总结共识并标注分歧。

## 失败处理

- 如果 ASP CLI 返回连接错误或超时，直接回复失败，提示用户运行 `asp doctor --output json`，检查 API URL、API key、网络连通性、用户状态和权限。不要尝试绕过认证。
- Provider 返回错误时，报告错误并建议检查 provider 配置。
- 指标格式无效时，说明该类型的期望格式。
- 没有可用 provider 时，直接说明。
