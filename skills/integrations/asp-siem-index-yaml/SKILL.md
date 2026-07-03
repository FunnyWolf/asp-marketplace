---
name: asp-siem-index-yaml
description: "创建或更新 ASP SIEM index YAML。适用于根据后端字段发现结果生成可维护的索引配置。"
argument-hint: "create SIEM index YAML"
compatibility: requires asp-cli >= 0.1.0
disable-model-invocation: true
metadata:
  author: Funnywolf
  version: 1.0.0
  cli: asp
  category: cyber security
  tags: [SIEM, YAML, schema, authoring]
  documentation: https://asp.viperrtp.com/
---

# ASP SIEM Index YAML

当用户要新建或更新 SIEM 索引配置 YAML 时，使用这个 skill，引导完成从字段发现到配置落盘的完整流程。

默认使用用户的语言回复；如果用户没有明确语言偏好，使用中文。

## 前置检查

如果 ASP CLI 尚未配置或认证状态不明确，先运行 `/asp-setup`。也可以直接执行：

```bash
asp doctor --output json
```

## 适用场景

- 用户想为某个 SIEM index 生成索引配置 YAML。
- 用户想根据后端实时字段更新现有 index YAML。
- 用户想查看某个 index 在 ELK 或 Splunk 中实际有哪些字段。
- 用户想为后续 SIEM 搜索、字段发现、规则编写或自适应查询准备字段元数据。

## 运行规则

- 配置文件固定存放在 `custom/data/siem/<index_name>.yaml`。
- 必须通过 `asp siem fields discover` 从后端拉取实时字段，不要跳过字段发现直接手写。
- `name`、`type`、`sample_values` 直接采用发现结果；`sample_values` 保持原始类型，不强转字符串。
- `description` 和 `is_key_field` 由模型根据字段语义和 sample values 推断，并标注为待确认。
- 不要在用户确认前覆盖现有 YAML。
- 有现有 YAML 时，以现有内容为基线，只推断新增字段或类型变化字段。

## 命令契约

字段发现命令：

```bash
asp siem fields discover <index_name> <backend> --from <start> --to <end> --doc-limit 10000 --max-samples-per-field 20 --output json
```

参数说明：

- `index_name`：SIEM 索引或数据源名称。
- `backend`：`ELK` 或 `Splunk`。
- `--from` / `--to`：采样时间范围，必须是带时区的 ISO 8601，例如 `2026-06-23T12:00:00Z`。
- `--doc-limit`：扫描日志条数，默认 10000。调小可加快返回，调大可提高样例覆盖率。
- `--max-samples-per-field`：每个字段返回的最大样例值数量，默认 20。

返回数据通常包含：

- `name`：字段名，嵌套字段使用点号路径。
- `type`：后端报告的字段类型。
- `sample_values`：样例值列表，保持原始类型。

## 决策流程

1. 如果缺少 `index_name`、`backend` 或采样时间范围，先询问。
2. 检查 `custom/data/siem/<index_name>.yaml` 是否已存在。
3. 已存在时读取为基线；不存在时标记为新建。
4. 调用 `asp siem fields discover` 获取实时字段。
5. 根据字段发现结果生成草案。
6. 展示差异摘要和关键字段列表，要求用户 review。
7. 用户确认后，写入 `custom/data/siem/<index_name>.yaml`。

## SOP

### Step 1 — 获取输入

要求用户提供：

- `index_name`：SIEM 索引名称。
- `backend`：`ELK` 或 `Splunk`。
- `time_range_start` 和 `time_range_end`：带时区的 ISO 8601 采样时间范围。

### Step 2 — 检查现有配置

检查 `custom/data/siem/<index_name>.yaml` 是否已存在。

- 如果存在，读取作为基线，后续展示差异。
- 如果不存在，按新建处理。

### Step 3 — 发现字段

执行：

```bash
asp siem fields discover <index_name> <backend> --from <start> --to <end> --max-samples-per-field 20 --output json
```

如果返回空字段列表，说明可能 index 名称有误、时间范围内无数据，或后端配置不可用。

### Step 4 — 处理 sample values

根据字段特征决定 `sample_values` 的合理数量：

| 字段特征 | 样例数量 | 判断依据 |
| --- | --- | --- |
| 时间戳类 | 1 | 类型为 date/timestamp，或值匹配 ISO-8601 / Unix epoch |
| 高基数长文本 | 1-2 | 值长度 > 32，或字段名含 uuid/hash/url/path/uri/raw/line |
| 布尔 / 状态开关 | 最多 2 | 类型为 boolean，或值仅为 true/false/0/1 |
| 低基数枚举 | 全量 | 去重后值数量 ≤ 10 |
| 中基数枚举 | 10 | 11-30 个不同值，取 Top-10 并注明总数 |
| 高基数枚举 | 5 | > 30 个不同值，取 Top-5 并注明总数 |
| 其他普通文本或数值 | 5 | 默认 Top-5 |

处理规则：

- 后端返回数量少于规则要求时，保持原样。
- 后端返回数量多于规则要求时，按规则裁剪并在草案中标注原因。
- 后端返回数量刚好符合时，直接采用。

### Step 5 — 生成草案

对每个字段填充：

| 字段 | 来源 |
| --- | --- |
| `name` | 字段发现结果 |
| `type` | 字段发现结果 |
| `sample_values` | 按自适应规则裁剪后采用 |
| `description` | 根据字段名语义和样例值推断 |
| `is_key_field` | 根据调查价值启发式推断 |

`is_key_field` 推断规则：

- `true`：身份、资产、网络四元组、动作、结果、高信号标识字段。例如 `src_ip`、`dst_ip`、`user`、`username`、`action`、`event_type`、`severity`、`process_name`、`file_hash`、`domain`。
- `false`：纯噪音或低价值元数据字段。例如 `_id`、`@version`、`beat.hostname`、`log.offset`、`input.type`、`agent.ephemeral_id`。

存在基线时：

- 保留已有字段的 `description` 和 `is_key_field`，除非字段类型发生变化。
- 只对新增字段或类型变化字段重新推断。
- 草案摘要中标注字段来源：`保留自基线`、`新推断` 或 `类型变化后重新推断`。

### Step 6 — 展示草案

先展示差异摘要和关键字段列表，不要直接倾倒完整 YAML。

首选回复结构：

**草案摘要**

- 目标索引：`<index_name>`
- 后端：`<backend>`
- 字段总数：`<n>`
- 新增字段：`<list>`
- 类型变化：`<list>`
- `is_key_field=true` 字段：`<list>`

**待确认项**

- `is_key_field` 推断是否合理。
- `description` 是否符合命名规范。
- 是否需要调整字段说明或样例值。

### Step 7 — 写入文件

用户确认后，将完整 YAML 写入：

```text
custom/data/siem/<index_name>.yaml
```

YAML 顶层结构：

```yaml
name: <index_name>
backend: <backend>
description: <根据 index_name 和字段整体语义生成>

fields:
  - name: <field_name>
    type: <field_type>
    description: <field_description>
    is_key_field: <true|false>
    sample_values: [<value1>, <value2>]
```

## 澄清规则

- 缺少 `index_name`、`backend` 或采样时间范围时，询问。
- `backend` 不是 `ELK` 或 `Splunk` 时，要求用户确认。
- 字段发现返回空字段列表时，要求用户确认 index 名称、时间范围和后端数据。
- 用户对某些字段的 `is_key_field` 或 `description` 有异议时，只调整受影响字段，不需要重新展示整个草案。

## 输出规则

- 草案展示优先使用差异摘要和关键字段列表。
- 只有用户确认写入时才生成完整 YAML 文件内容。
- 保持简洁，不要逐个解释所有字段。
- 明确区分从后端发现的字段和模型推断的说明。

## 失败处理

- 如果 ASP CLI 返回连接错误或超时，直接回复失败，提示用户运行 `asp doctor --output json`，检查 API URL、API key、网络连通性、用户状态和权限。不要尝试绕过认证。
- 如果字段发现失败，说明错误并要求用户检查 index 名称、时间范围和后端连通性。
- 如果返回字段数为 0，提示用户确认 index 是否存在且采样范围内有数据。
- 如果用户要求写入但未经过 review 确认，提醒先完成确认步骤。
