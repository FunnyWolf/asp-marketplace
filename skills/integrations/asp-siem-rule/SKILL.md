---
name: asp-siem-rule
description: "编写并验证 ASP SIEM 检测规则。适用于将威胁场景转化为 Splunk SPL 或 ELK ES|QL 查询。"
argument-hint: "create detection rule <threat scenario>"
compatibility: requires asp-cli >= 0.1.0
disable-model-invocation: true
metadata:
  author: Funnywolf
  version: 1.0.0
  cli: asp
  category: cyber security
  tags: [SIEM, detection, rule, SPL, ESQL, MITRE]
  documentation: https://asp.viperrtp.com/
---

# ASP SIEM Rule

当用户需要编写 SIEM 检测规则时，使用这个 skill，引导完成从威胁场景描述到可验证检测规则的完整流程。

默认使用用户的语言回复；如果用户没有明确语言偏好，使用中文。

## 前置检查

如果 ASP CLI 尚未配置或认证状态不明确，先运行 `/asp-setup`。也可以直接执行：

```bash
asp doctor --output json
```

## 适用场景

- 用户想为某个威胁场景创建 SIEM 检测规则。
- 用户需要编写 Splunk SPL 或 ELK ES|QL 告警查询。
- 用户想把安全检测逻辑转化为可部署的 SIEM 规则。
- 用户想了解如何用 SPL 或 ES|QL 实现特定检测逻辑。
- 用户已有部分查询语句，需要完善、验证或解释。

## 运行规则

- 必须先确认目标平台是 Splunk 还是 ELK，不得假设。
- 编写规则前必须探索 schema，了解可用索引和字段。
- 规则生成后必须通过 ASP CLI 验证；除非验证不可行，否则不要输出未经验证的最终规则。
- 验证必须提供带时区的 ISO 8601 时间范围。
- MITRE ATT&CK 映射必须基于实际检测行为，不得随意编造。
- 优化目标是生成可直接部署和调优的检测规则，而不是理论查询示例。

## 命令契约

### 探索 schema

```bash
asp siem schema list --output json
asp siem schema show <index_name> --output json
```

### 验证 SPL

```bash
asp siem query spl "<query>" --from <start> --to <end> --limit 100 --output json
```

### 验证 ES|QL

```bash
asp siem query esql "<query>" --from <start> --to <end> --limit 100 --output json
```

要求：

- `--from` 和 `--to` 必须是带时区的 ISO 8601。
- `--limit` 范围 1-10000。
- 查询验证结果用于调试字段、阈值和误报，不等同于正式部署结果。

## 决策流程

1. 确认用户使用 Splunk 还是 ELK。
2. 调用 `asp siem schema list --output json` 获取可用索引，帮助用户选择目标索引。
3. 调用 `asp siem schema show <index_name> --output json` 获取字段详情。
4. 确认用户想检测的威胁场景。
5. 确认检测窗口和验证时间范围。
6. 根据检测目标和可用字段生成 SPL 或 ES|QL。
7. 调用 `asp siem query spl` 或 `asp siem query esql` 验证规则。
8. 根据验证结果调整查询、阈值和误报排除条件。
9. 只有行为明确时才映射 MITRE ATT&CK。
10. 输出最终规则和使用说明。

## SOP

### Step 1 — 确认 SIEM 类型

询问用户使用 Splunk 还是 ELK。

- 明确回答 Splunk 时，生成 SPL。
- 明确回答 ELK 时，生成 ES|QL。
- 用户不确定时，询问他们平时在哪个平台查看告警或日志。

### Step 2 — 探索 schema

1. 调用 `asp siem schema list --output json` 获取所有索引列表。
2. 展示与检测目标最相关的候选索引。
3. 调用 `asp siem schema show <index_name> --output json` 获取目标索引字段详情。
4. 总结关键字段分类：
   - 时间字段，例如 `@timestamp`、`eventTime`。
   - 主体字段，例如用户名、IP、主机名。
   - 目标字段，例如目标用户、目标资源。
   - 结果字段，例如状态码、错误信息、成功/失败。
   - 网络字段，例如源 IP、目标 IP、端口。
5. 推荐适合检测的字段组合。

### Step 3 — 确认检测目标

引导用户明确：

- 检测什么行为，例如暴力破解、异常登录、权限提升、可疑进程、横向移动。
- 关键判断条件，例如失败次数阈值、异常时间范围、来源国家、进程路径。
- 统计窗口，例如 5 分钟、1 小时、24 小时。
- 验证时间范围，即实际运行查询的 `--from` 和 `--to`。

用户描述模糊时，优先问一个聚焦问题：

- “你想检测什么类型的异常行为？”
- “这个行为最关键的字段特征是什么？”
- “你希望在多长时间窗口内触发？”

### Step 4 — 生成检测规则

Splunk SPL 常见结构：

```spl
index=<index_name> <filter_conditions>
| stats count by <aggregation_fields>
| where count > <threshold>
```

ELK ES|QL 常见结构：

```esql
FROM <index_name>
| WHERE <filter_conditions>
| STATS count = COUNT() BY <aggregation_fields>
| WHERE count > <threshold>
```

生成规则时考虑：

- 时间过滤条件。
- 字段匹配条件。
- 聚合逻辑，例如按用户、源 IP、主机聚合。
- 阈值设置。
- 排除误报的条件。
- 字段是否实际存在于 schema 中。

### Step 5 — 验证规则

1. 调用 `asp siem query spl` 或 `asp siem query esql` 执行生成的规则。
2. 分析返回结果：
   - 是否有命中记录。
   - 命中数量是否合理。
   - 结果是否符合检测预期。
3. 无命中时：
   - 检查时间范围是否合适。
   - 检查字段名是否正确。
   - 适当放宽条件后重新验证。
4. 命中过多时：
   - 添加更多过滤条件。
   - 提高阈值。
   - 缩小时间范围。
5. 持续迭代直到结果合理，或明确说明为什么无法验证。

### Step 6 — MITRE ATT&CK 映射

根据检测行为映射 MITRE ATT&CK：

- 识别检测行为所属战术，例如 Initial Access、Execution、Persistence、Credential Access。
- 确定具体技术，例如 T1110 Brute Force、T1078 Valid Accounts。
- 一个规则可能映射到多个技术。
- 如果不确定，说明映射理由和不确定性。

不要仅凭规则标题或用户描述编造映射；映射必须来自实际检测行为。

### Step 7 — 输出最终规则

最终输出必须包含：

1. **规则查询语句**：可复制到 SIEM 中使用。
2. **验证结果摘要**：验证时间范围、命中情况、代表性结果或无命中原因。
3. **MITRE ATT&CK 映射**：仅在有依据时输出。
4. **使用说明**：部署方式、建议执行频率、阈值调整建议、可能误报和排除方法。

## 澄清规则

- 用户未说明 SIEM 类型时，先询问。
- 用户未提供检测目标时，先询问。
- 目标索引不确定时，使用 schema 探索帮助选择。
- 字段含义不明确时，基于 schema 和样例值解释。
- 用户已有部分查询语句时，在此基础上完善，不要无故重写。
- 缺少验证时间范围时，先询问。

## 输出规则

- 规则代码使用代码块，方便复制。
- 每个步骤只总结关键结果，不罗列所有字段。
- 验证结果重点展示命中情况和代表性记录。
- 最终规则必须包含完整可执行查询语句。
- 使用说明简洁实用，不写理论性长文。
- 如果无法验证，必须明确标注“未验证”，并说明原因。

## 失败处理

- 如果 ASP CLI 返回连接错误或超时，直接回复失败，提示用户运行 `asp doctor --output json`，检查 API URL、API key、网络连通性、用户状态和权限。不要尝试绕过认证。
- 如果 schema 列表为空，说明没有配置索引，提示用户先完成索引配置。
- 如果规则验证无结果，检查时间范围、字段名和条件是否过严。
- 如果规则验证结果过多，建议添加过滤条件、提高阈值或缩小时间范围。
- 如果现有字段无法支持该威胁场景，说明原因并建议可行替代检测方向。
