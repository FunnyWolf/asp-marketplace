---
name: asp-siem-rule-zh
description: '帮助用户编写 Splunk SPL 或 ELK ES|QL 检测规则。当用户想创建 SIEM 检测规则、编写告警查询、设计安全检测逻辑时使用。'
argument-hint: '<threat-scenario>'
compatibility: connect to asp mcp server
metadata:
  author: Funnywolf
  version: 0.1.0
  mcp-server: asp
  category: cyber security
  tags: [ SIEM, detection, rule, SPL, ES|QL, MITRE ]
  documentation: https://asp.viperrtp.com/
---

# ASP SIEM Rule Creator

当用户需要编写 SIEM 检测规则时，使用这个 skill 引导完成从威胁场景描述到可用检测规则的完整流程。

## 适用场景

- 用户想为某个威胁场景创建 SIEM 检测规则。
- 用户需要编写 Splunk SPL 或 ELK ES|QL 告警查询。
- 用户想把安全检测逻辑转化为可部署的 SIEM 规则。
- 用户想了解如何用 SPL/ES|QL 实现特定检测逻辑。

## 运行规则

- 必须先确认用户使用 Splunk 还是 ELK，不得假设。
- 编写规则前必须先探索 schema，了解可用索引和字段。
- 规则生成后必须通过 SIEM API 验证，不得输出未经验证的规则。
- MITRE ATT&CK 映射必须基于实际检测行为，不得随意编造。
- 优化目标是生成可直接部署使用的检测规则，而不是理论性的查询示例。

## 决策流程

1. 询问用户使用 Splunk 还是 ELK。
2. 调用 `siem_explore_schema()` 获取可用索引，帮助用户选择目标索引。
3. 调用 `siem_explore_schema(target_index=<index>)` 获取字段详情。
4. 询问用户想检测什么威胁场景。
5. 根据检测目标和可用字段，生成 SPL 或 ES|QL 检测规则。
6. 调用 `siem_execute_spl` 或 `siem_execute_esql` 验证规则。
7. 根据验证结果调整规则。
8. 映射 MITRE ATT&CK 战术和技术。
9. 输出最终规则和使用说明。

## SOP

### Step 1 — 确认 SIEM 类型

询问用户使用 Splunk 还是 ELK。

- 如果用户明确回答，记录选择并继续。
- 如果用户不确定，询问他们平时在哪个平台查看告警。
- 选择 Splunk 将生成 SPL 规则，选择 ELK 将生成 ES|QL 规则。

### Step 2 — 探索 Schema

1. 调用 `siem_explore_schema()` 获取所有索引列表。
2. 展示可用索引，帮助用户选择与检测目标相关的索引。
3. 调用 `siem_explore_schema(target_index=<index>)` 获取目标索引的字段详情。
4. 总结关键字段分类：
   - 时间字段（如 `@timestamp`、`eventTime`）
   - 主体字段（用户名、IP、主机名等）
   - 目标字段（目标用户、目标资源等）
   - 结果字段（状态码、错误信息等）
   - 网络字段（源IP、目标IP、端口等）
5. 推荐适合检测的字段组合。

### Step 3 — 确认检测目标

询问用户想检测什么威胁场景。引导用户明确：

- 检测什么行为（如暴力破解、异常登录、权限提升等）
- 关键判断条件（如失败次数阈值、异常时间范围等）
- 时间窗口（如过去 1 小时、过去 24 小时等）

如果用户描述模糊，通过以下问题引导：
- "你想检测什么类型的异常行为？"
- "这个行为的关键特征是什么？"
- "你希望在多长时间范围内检测？"

### Step 4 — 生成检测规则

根据检测目标和可用字段，生成检测规则：

**Splunk SPL 规则结构：**
```
index=<index_name> <filter_conditions>
| stats count by <aggregation_fields>
| where count > <threshold>
```

**ELK ES|QL 规则结构：**
```
FROM <index_name>
| WHERE <filter_conditions>
| STATS count = COUNT() BY <aggregation_fields>
| WHERE count > <threshold>
```

生成规则时考虑：
- 时间过滤条件
- 字段匹配条件
- 聚合逻辑（如按用户、IP、主机聚合）
- 阈值设置
- 排除误报的条件

### Step 5 — 验证规则

1. 调用 `siem_execute_spl` 或 `siem_execute_esql` 执行生成的规则。
2. 分析返回结果：
   - 是否有命中记录
   - 命中数量是否合理
   - 结果是否符合检测预期
3. 如果无命中：
   - 检查时间范围是否合适
   - 检查字段名是否正确
   - 适当放宽条件后重新验证
4. 如果命中太多：
   - 添加更多过滤条件
   - 提高阈值
   - 缩小时间范围
5. 持续迭代直到结果合理。

### Step 6 — MITRE ATT&CK 映射

根据检测的威胁行为，映射到 MITRE ATT&CK：

- 识别检测的行为属于哪个战术（如 Initial Access、Execution、Persistence 等）
- 确定具体技术（如 T1110 - Brute Force、T1078 - Valid Accounts 等）
- 输出格式：`Tactic: xxx | Technique: xxx (Txxxx)`

映射时注意：
- 基于实际检测行为映射，不要猜测
- 一个规则可能映射到多个技术
- 如果不确定，说明映射理由

### Step 7 — 输出最终规则

输出包含：

1. **规则查询语句**：可直接粘贴到 SIEM 中使用
2. **MITRE ATT&CK 映射**：战术和技术编号
3. **使用说明**：
   - 如何部署到 SIEM（如 Splunk saved search、ELK alerting rule）
   - 建议的执行频率
   - 阈值调整建议
   - 可能的误报场景和排除方法

## 澄清规则

- 如果用户未说明 SIEM 类型，必须先询问。
- 如果用户未提供检测目标，必须先询问。
- 如果用户对目标索引不确定，使用 `siem_explore_schema` 帮助选择。
- 如果用户对字段含义有疑问，基于 schema 返回的样本值解释。
- 如果用户已经有部分查询语句，在此基础上完善而不是重新生成。

## 输出规则

- 规则代码使用代码块格式，方便复制。
- 每个步骤的结果简洁总结，不罗列所有字段。
- 验证结果重点展示命中情况和代表性记录。
- 最终规则必须包含完整的可执行查询语句。
- 使用说明简洁实用，不包含理论性内容。

## 失败处理

- 如果 MCP 工具调用返回连接错误或超时，直接回复失败，提示用户检查 `ASP_MCP_SSE_URL` 环境变量是否已配置，并确认 ASP MCP 服务器已启动。不要尝试重试或绕过。
- 如果 `siem_explore_schema` 返回空索引列表，说明没有配置索引，提示用户先完成索引配置。
- 如果规则验证无结果，检查：时间范围是否合适、字段名是否正确、条件是否过严。
- 如果规则验证结果过多，建议：添加更多过滤条件、提高阈值、缩小时间范围。
- 如果用户提供的威胁场景无法用现有字段检测，说明原因并建议可用的替代方案。
