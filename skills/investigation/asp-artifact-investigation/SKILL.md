---
name: asp-artifact-investigation
description: "以 artifact 或 IOC 为主线编排 ASP 调查、上下文补充和 pivot 选择。"
argument-hint: "investigate artifact <artifact_id or value>"
compatibility: requires asp-cli >= 0.1.0
disable-model-invocation: true
metadata:
  author: Funnywolf
  version: 1.0.0
  cli: asp
  category: cyber security
  tags: [artifact, IOC, investigation, pivot]
  documentation: https://asp.viperrtp.com/
---

# ASP Artifact Investigation

当用户要以 artifact 或 IOC 为起点调查其意义、上下文、范围或下一步时，使用这个用户显式调用的编排 skill。目标是选择最少、最高价值的 pivot，并在证据足够时停止。

默认使用用户的语言回复；如果用户没有明确语言偏好，使用中文。

## 前置检查

如果 ASP CLI 尚未配置或认证状态不明确，先运行 `/asp-setup`。也可以直接执行：

```bash
asp doctor --output json
```

## 何时使用

- 用户要调查某个 IOC 或 artifact 的意义、范围、上下文或下一步。
- 用户要围绕 IP、域名、hash、URL、用户名、主机名等可观察对象做受控调查。
- 用户需要在 artifact、SIEM、CMDB、威胁情报、knowledge、enrichment、comment，以及条件性的 alert/case follow-up 之间选择最有价值的 pivot。

## 不要使用

- 用户只是要查单个 artifact、列出 artifact 或做单步对象查询，应使用 `asp-artifact`。
- 用户已经明确指定底层操作，不需要多步调查编排。
- 当前问题本质上是 case 主导，应使用 `asp-case-investigation`。
- 当前问题是范围化威胁狩猎，应使用 `asp-threat-hunting`。

## 编排原则

- Artifact 或 IOC 是默认主视图。
- 先判断该对象是否已经在 ASP 中存在为 artifact；如果不存在，也可以继续做 IOC 主导调查。
- SIEM 只用于回答“出现在哪里、频率如何、时间上怎么分布、周边对象是什么”。
- CMDB 只用于回答内部归属、资产、身份或业务上下文。
- 威胁情报只用于外部 IOC 声誉和威胁上下文。
- Knowledge 只用于解释背景、模式、误报经验或环境特定处理。
- Enrichment 只用于保存稳定结构化结论，不用于临时笔记。
- Alert / case 只作为条件性 follow-up，不作为默认可取上下文。
- 默认最多做一到两个高价值 pivot；除非用户明确要求深挖，否则不要扩展成广泛 hunt。

## 可编排能力

- `asp-artifact`：artifact 查询、artifact 审查、对象级上下文。
- `asp-siem-search`：IOC 检索、流行度、时间线、周边活动。
- `asp-cmdb`：资产、身份、负责人和业务上下文。
- `asp-threat-intelligence`：IOC 声誉和外部威胁上下文。
- `asp-knowledge`：内部上下文、已知模式、处理建议。
- `asp-enrichment`：持久化结构化调查结论。
- `asp-comment`：自然语言分析师备注或交接评论。
- `asp-file`：评论附件元数据、下载和文本读取。
- `asp-alert`：条件性的 alert follow-up。
- `asp-case`：条件性的 case follow-up。
- `asp-playbook`：自动化建议或自动化历史。

## 命令入口

### 已有 artifact ID

```bash
asp artifact show <artifact_id> --include-related --output json
```

### 原始 IOC 或对象值

```bash
asp artifact list --value <value> --include-related --output json
```

### 威胁情报上下文

```bash
asp ti query <indicator> --artifact-type <type> --output json
```

### CMDB 上下文

```bash
asp cmdb lookup <artifact_type> <artifact_value> --output json
```

### SIEM 证据

```bash
asp siem search keyword "<keyword>" --from <start> --to <end> --output json
```

SIEM 查询必须有明确时间范围。缺少时间范围时，先询问最窄可行范围。

## 推荐流程

### 1. 从 artifact 或 IOC 开始

1. 用户给 artifact ID 时，先调用 `asp artifact show <artifact_id> --include-related --output json`。
2. 用户给 IOC 值时，先调用 `asp artifact list --value <value> --include-related --output json`。
3. 如果没有匹配 artifact，也继续以 IOC 为中心调查，不要假装 artifact 记录已经存在。
4. 总结值、类型、角色、是否为平台已有记录，以及当前为什么重要或为什么还不够重要。

### 2. 判断威胁情报是否有价值

适合使用威胁情报的情况：

- 对象是 IP、domain、URL、hash、邮箱等可外部查询 IOC。
- 用户问“是否恶意”“有没有声誉”“外部情报怎么看”。
- 需要为 case 或 artifact 提供外部风险上下文。

威胁情报只能作为辅助上下文，不能替代环境内证据。

### 3. 判断 CMDB 是否有价值

适合使用 CMDB 的情况：

- 对象可能属于内部资产、账号、邮箱、主机、资源、子网、端口或设备。
- 用户需要 owner、业务系统、资产重要性、身份或负责人信息。
- 风险判断依赖业务影响面。

CMDB 用于解释内部上下文，不用于证明恶意性。

### 4. 判断 SIEM 是否合理

只有当需要知道以下内容时才使用 SIEM：

- IOC 在哪里出现。
- 出现频率和历史分布。
- 首次或最近出现时间。
- 周边对象、相关用户、主机或进程。
- 是否影响多个资产或 case。

如果 IOC 太弱、太宽泛或缺少时间范围，先指出限制并收窄计划。

### 5. 判断 knowledge 是否合理

当内部经验、误报模式、处理建议或环境特定上下文可能改变判断时，才查 knowledge。

- 返回小而相关的候选列表。
- 不做广泛检索。

### 6. 判断父层 follow-up 是否合理

只有当当前上下文显式暴露出 alert 或 case 路径时，才建议继续到父层。

- 如果 artifact 查询返回相关 alert，可建议查看最相关 alert。
- 如果 alert 指向 case，可建议进入 case 调查。
- 如果看不到父层关系，明确说明，不要补全想象中的上下文。

### 7. 判断是否需要保存结果

只有当已经形成稳定结构化结论时，才建议 enrichment。

- 用户明确要求保存结构化结果时，使用 `asp-enrichment`。
- 用户明确要求添加备注或交接说明时，使用 `asp-comment`。
- 默认不持久化。

## 停止条件

- 已经足以回答 IOC / artifact 的意义和下一步。
- 继续扩展只会增加噪音，不会明显提升判断质量。
- 缺少关键输入，无法继续有效搜索。
- 当前平台能力不支持继续下钻。
- 用户只要求单步操作，已转交底层 skill。

## 输出要求

默认输出结构：

- `Artifact 理解`：说明可观察对象是什么，以及当前为什么重要或不重要。
- `已知上下文`：artifact 事实、TI、CMDB、alert/case 关联等已知信息。
- `最佳 pivot`：最值得做的一到三个 pivot；没有就明确写“无”。
- `证据缺口`：仍需确认的范围、时间线、上下文或数据源。
- `建议下一步`：最多三条，必须具体且受当前能力支持。

输出时必须：

- 区分观察到的事实、重要性判断和推荐 pivot。
- 不倾倒原始 JSON。
- 不把每个 artifact 都扩展成完整 hunt。
- 不凭 artifact 存在就声称它恶意。

## 特殊情况处理

- 找不到 artifact：直接说明。
- 用户只提供原始 IOC 而不是现有 artifact：继续面向查询的调查，不假装 artifact 记录已存在。
- 看不到支持的父 alert 或 case 关系：清楚说明。
- IOC 太广泛或模糊：解释限制，并提出最窄有用的下一个 pivot。
- 用户要求持久化：根据内容选择 enrichment 或 comment，不发明自定义保存路径。

## 失败处理

- 如果 ASP CLI 返回连接错误或超时，直接回复失败，提示用户运行 `asp doctor --output json`，检查 API URL、API key、网络连通性、用户状态和权限。不要尝试绕过认证。
- 如果 artifact 不存在，直接说明。
- 如果 SIEM 缺少时间范围，询问最窄可行范围。
- 如果 TI、CMDB 或 SIEM 查询失败，说明失败点，不要假装该上下文存在。
