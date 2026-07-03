---
name: asp-cmdb
description: "查询 ASP CMDB 上下文。适用于获取资产、身份、负责人、业务系统和内部环境信息。"
argument-hint: "lookup <artifact_type> <artifact_value>"
compatibility: requires asp-cli >= 0.1.0
metadata:
  author: Funnywolf
  version: 1.0.0
  cli: asp
  category: cyber security
  tags: [CMDB, asset, identity, owner, business-context]
  documentation: https://asp.viperrtp.com/
---

# ASP CMDB

当用户需要为可观测对象查询业务、资产、身份、邮箱、资源、负责人或关联上下文时，使用这个 skill。

默认使用用户的语言回复；如果用户没有明确语言偏好，使用中文。

## 前置检查

如果 ASP CLI 尚未配置或认证状态不明确，先运行 `/asp-setup`。也可以直接执行：

```bash
asp doctor --output json
```

## 适用场景

- 用户想知道某个 IP、主机名、账号、邮箱、资源、子网、端口或设备属于谁。
- 用户在分诊前需要 owner、业务系统、资产重要性或身份上下文。
- Artifact 调查除了 SIEM 证据或威胁情报外，还需要内部环境上下文。
- 用户想把有用的 CMDB 上下文保存到 case、alert 或 artifact。

## 运行规则

- CMDB 用于解释内部归属、业务语义和影响面，不用于证明恶意性。
- 如果 artifact 已存在于 ASP，优先使用 artifact 记录中的 type 和 value。
- 如果查询结果会影响风险判断，明确说明影响点。
- 如果需要保存 CMDB 上下文，转交 `asp-enrichment`。
- 声誉、恶意性或外部 IOC 风险应使用 `asp-threat-intelligence`，不要用 CMDB 替代。

## 命令契约

```bash
asp cmdb lookup "IP Address" 8.8.8.8 --output json
```

指定 provider：

```bash
asp cmdb lookup "User Name" alice --provider internal-cmdb --output json
```

参数说明：

- `artifact_type` 应使用 ASP artifact 类型，例如 `IP Address`、`Hostname`、`User Name`、`Email Address`、`Resource UID`、`Port`、`Subnet`、`Account`、`Resource`、`Device`、`Other`。
- `artifact_value` 是要查询的精确值。
- `provider` 可选。省略时查询所有已配置 CMDB provider。
- 返回内容可能包括 `asset`、`identity`、`mailbox`、`resource`、`port`、`subnet`、`business`、`owner`、`related`、`raw` 和 `error`。

## 决策流程

1. 从用户请求或当前 artifact 上下文提取 `artifact_type` 和 `artifact_value`。
2. 如果类型有歧义，选择最可能的 ASP artifact type，并明确说明该假设。
3. 调用 `asp cmdb lookup <artifact_type> <artifact_value> --output json`。
4. 只总结会影响调查判断的上下文。
5. 如果用户要保存结果，转交 `asp-enrichment`。

## SOP

### 查询 CMDB

1. 确认 `artifact_value`。
2. 推断或确认 `artifact_type`。
3. 除非用户明确要求，不要询问 provider。
4. 执行查询。
5. 汇总 owner、业务系统、资产重要性、身份上下文和有价值关联对象。
6. 用一句话说明这些上下文如何影响分诊或下一步 pivot。

首选回复结构：

**CMDB Context**

- `Observable`：artifact type 和 value。
- `Provider`：返回上下文的 provider。
- `Owner / Business`：负责人、业务系统、重要性。
- `Asset or Identity`：资产或身份详情。
- `Related`：有价值的关联对象。

**Assessment**

- 用一句话说明该上下文如何影响分诊、影响面或下一步 pivot。

## 澄清规则

- 缺少 `artifact_value` 时，询问。
- 无法从值或当前 artifact 合理推断 `artifact_type` 时，询问。
- 不要询问 provider，除非用户明确要求选择 provider 或多 provider 结果冲突。

## 输出规则

- 保持简洁。
- 除非用户要求，否则不要倾倒原始 JSON。
- 区分 CMDB 事实和调查判断。
- 如果返回多个候选结果，说明歧义，不要擅自选择唯一对象。

## 失败处理

- 如果 ASP CLI 返回连接错误或超时，直接回复失败，提示用户运行 `asp doctor --output json`，检查 API URL、API key、网络连通性、用户状态和权限。不要尝试绕过认证。
- 如果 provider 返回错误，报告错误并建议检查 provider 配置。
- 如果没有 provider 支持该 artifact type，直接说明。
- 如果无结果，说明没有可用 CMDB 上下文，并继续使用已有证据。
