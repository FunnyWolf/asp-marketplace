---
name: asp-case-investigation
description: "以 case 为主线编排 ASP 调查、分诊、证据审查和下一步建议。"
argument-hint: "investigate case <case_id>"
compatibility: requires asp-cli >= 0.1.0
disable-model-invocation: true
metadata:
  author: Funnywolf
  version: 1.0.0
  cli: asp
  category: cyber security
  tags: [case, investigation, triage, orchestration]
  documentation: https://asp.viperrtp.com/
---

# ASP Case Investigation

当用户要围绕 ASP case 做调查、分诊、证据判断或下一步决策时，使用这个用户显式调用的编排 skill。职责不是重复底层 skill，而是围绕用户问题选择最少、最高价值的调查路径，并在证据足够时停止。

默认使用用户的语言回复；如果用户没有明确语言偏好，使用中文。

## 前置检查

如果 ASP CLI 尚未配置或认证状态不明确，先运行 `/asp-setup`。也可以直接执行：

```bash
asp doctor --output json
```

## 何时使用

- 用户要理解、审查、分诊或调查某个 case。
- 用户要判断当前 case 的证据是否足够、风险是否明确、下一步该做什么。
- 用户需要围绕 case 在 alert、artifact、SIEM、knowledge、enrichment、comment、CMDB、playbook 之间做受控编排。
- 用户提出的问题是 case 主导，而不是单个 IOC、单条 alert 或纯 SIEM hunt 主导。

## 不要使用

- 用户只是要列出、查看、更新单个 case 或单个 alert，这类请求应使用底层 skill。
- 用户已经明确指定底层操作，不需要多步调查编排。
- 当前问题本质上是 artifact 或 IOC 主导，应使用 `asp-artifact-investigation`。
- 当前问题本质上是范围化威胁狩猎，应使用 `asp-threat-hunting`。

## 编排原则

- Case 是默认主视图，先回答用户关于 case 的核心问题。
- 只有当 case 本身不足以回答问题时，才扩展到 alert、artifact、SIEM、knowledge、CMDB 或 playbook。
- 默认只做最小调查，不为了“完整性”而查询所有层。
- 默认最多做一到两个高价值 pivot；除非用户明确要求深挖，否则不要继续扩展。
- Enrichment 是调查产物，不是默认调查步骤。
- Comment 用于自然语言交接备注，不用于结构化证据。
- Playbook 属于行动或自动化阶段，只有问题已经进入执行阶段时才建议。
- 始终区分已知事实、分析判断和建议动作。

## 可编排能力

- `asp-case`：case 审查、AI 字段更新、case 级主视图。
- `asp-alert`：告警审查与检测上下文。
- `asp-artifact`：对象级查询和 artifact 上下文审查。
- `asp-siem-search`：证据检索、时间线扩展、范围界定、流行度判断。
- `asp-knowledge`：内部经验、处理建议、历史模式。
- `asp-enrichment`：持久化结构化调查结论。
- `asp-comment`：自然语言分析师备注或交接评论。
- `asp-file`：评论附件元数据、下载和文本读取。
- `asp-cmdb`：资产、身份、负责人和业务上下文。
- `asp-playbook`：自动化历史、template 查询或执行建议。

## 命令入口

### 获取 case 主视图

```bash
asp case show <case_id> --include-related --output json
```

### 获取 case 评论

仅当评论可能影响判断或用户明确要求查看评论时执行：

```bash
asp comment list <case_id> --output json
```

### 获取具体 alert 详情

```bash
asp alert show <alert_id> --include-related --output json
```

### 从对象继续下钻

```bash
asp artifact show <artifact_id> --include-related --output json
asp ti query <indicator> --artifact-type <type> --output json
asp cmdb lookup <artifact_type> <artifact_value> --output json
```

### 需要 SIEM 证据时

```bash
asp siem search keyword "<keyword>" --from <start> --to <end> --output json
```

SIEM 查询必须有明确时间范围。缺少时间范围时，先询问最窄可行范围。

## 推荐流程

### 1. 从 case 开始

1. 调用 `asp case show <case_id> --include-related --output json`。
2. 提炼状态、严重性、confidence、verdict、时间线、summary、AI 字段和最明显的证据缺口。
3. 如果用户问题可能受评论影响，再调用 `asp comment list <case_id> --output json`。
4. 如果找不到 case，直接说明，不继续猜测。

### 2. 判断是否需要 alert 上下文

只有当 case 本身不足以解释调查原因、触发检测或关键实体时，才补 alert 上下文。

- 只保留最相关的 alert，不做全量罗列。
- 如果 case 已经足以回答用户问题，不要为了完整性继续查 alert。
- 如果 alert 暴露出关键 artifact，再判断是否值得 pivot。

### 3. 识别 pivot 候选

只有当 case 或 alert 已经暴露出明确对象，且该对象会改变判断时，才向 artifact 或 IOC 层继续。

高价值 pivot 示例：

- 可疑源 IP、目标 IP、域名、URL、hash、用户名、主机名。
- 告警规则中出现的关键资源或云账号。
- 与风险判断直接相关的业务资产或身份。

如果没有具体对象，就明确说明无法继续向对象层下钻。

### 4. 判断是否需要 SIEM

只有当需要验证以下内容时才使用 SIEM：

- 事件时间线。
- 影响范围。
- 流行度或历史出现情况。
- 周边活动。
- 现有 evidence 是否支持或削弱判断。

缺少时间范围时，停止并询问最窄可行时间范围。不要默认做无界搜索。

### 5. 判断是否需要 knowledge

当已有模式、误报经验、处理建议或环境特定上下文可能显著影响判断时，才查询 knowledge。

- 返回小而相关的候选列表。
- 不做广泛检索。
- 如果 knowledge 改变建议，明确说明改变点。

### 6. 判断是否需要保存结果

只有当已经形成稳定的结构化结论，才建议 enrichment。

- 用户明确要求“保存结构化结果”时，使用 `asp-enrichment`。
- 用户明确要求“添加备注/交接说明”时，使用 `asp-comment`。
- 默认不持久化。

### 7. 推荐后续操作

只基于当前证据推荐一到三个具体后续动作。

- 证据不足时，推荐最小补证动作。
- 风险明确时，推荐处置或自动化方向。
- 问题已进入执行阶段时，才建议 playbook。

## 停止条件

- 已经足以回答用户问题。
- 继续扩展只会增加噪音，不会明显提升判断质量。
- 缺少关键输入，无法继续有效调查。
- 当前平台能力不支持继续下钻。
- 用户只要求单步操作，已转交底层 skill。

## 输出要求

默认输出结构：

- `Case 理解`：用一个简短段落说明该 case 似乎代表什么。
- `当前信号`：从 case、alert、comment、artifact 上下文中已知的关键事实。
- `有价值的 pivot`：真正值得做的下一步 pivot；没有就明确写“无”。
- `证据缺口或 SIEM 需求`：仍需确认或范围界定的内容。
- `知识库或复用线索`：仅当检查过相关 knowledge 时输出。
- `建议下一步`：最多三条，必须具体且受当前能力支持。

输出时必须：

- 区分已知事实、分析判断和建议动作。
- 不倾倒原始 JSON。
- 不把所有关联对象机械列完。
- 不使用看似确定但没有数据支持的措辞。

## 特殊情况处理

- 找不到 case：直接说明。
- 通过当前支持的 pivot 无法获得相关 alert 上下文：说明限制，并继续基于已知内容回答。
- 没有足够具体的 artifact pivot：不要发明一个。
- 用户在证据不足时要求最终判定：解释 confidence 差距。
- 用户要求的操作属于较低层 skill：调用或遵循该 skill 的流程，不重写工作流。
- 用户要求保存结果：先判断 comment 还是 enrichment，再执行对应 skill。

## 失败处理

- 如果 ASP CLI 返回连接错误或超时，直接回复失败，提示用户运行 `asp doctor --output json`，检查 API URL、API key、网络连通性、用户状态和权限。不要尝试绕过认证。
- 如果 case 不存在，直接说明。
- 如果 SIEM 缺少时间范围，询问最窄可行范围。
- 如果某个 pivot 查询失败，说明失败点，不要假装该上下文存在。
