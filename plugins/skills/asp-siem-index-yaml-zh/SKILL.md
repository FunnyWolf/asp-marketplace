---
name: asp-siem-index-yaml-zh
description: '新建或更新 SIEM 索引配置 YAML。当用户提到 SIEM index、字段配置、索引 schema、ELK/Splunk 字段发现、或想把后端字段同步到 YAML 配置文件时，主动使用此 skill——即使用户没有明确说"生成 YAML"。'
argument-hint: '<index_name> <backend>'
compatibility: connect to asp mcp server
metadata:
  author: Funnywolf
  version: 0.2.0
  mcp-server: asp
  category: cyber security
  tags: [ SIEM, index, yaml, schema, configuration ]
  documentation: https://asp.viperrtp.com/
---

# ASP SIEM Index YAML

当用户要新建或更新 SIEM 索引配置 YAML 时，使用这个 skill 引导完成从字段发现到配置落盘的完整流程。

## 适用场景

- 用户想为某个 SIEM index 生成索引配置 YAML。
- 用户想根据后端实时字段更新现有索引 YAML。
- 用户想查看某个 index 在 ELK 或 Splunk 中实际有哪些字段。

## 运行规则

- 配置文件固定存放在 `DATA/Plugin_SIEM_Indexes/<index_name>.yaml`。
- 必须通过 `siem_discover_index_fields` 从后端拉取实时字段，不得跳过直接手写。
- `name`、`type`、`sample_values` 直接采用发现结果，`sample_values` 保持原始类型，不强转字符串。
- `description` 和 `is_key_field` 由模型根据字段语义和 sample_values 推断，标注为待确认。
- 不要在用户确认前覆盖现有 YAML。

## 决策流程

1. 如果用户未提供 `index_name` 或 `backend`，先询问。
2. 如果 `DATA/Plugin/SIEM/<index_name>.yaml` 已存在，读取并作为对比基线。
3. 调用 `siem_discover_index_fields` 获取实时字段。
4. 生成草案，展示给用户 review。
5. 用户确认后写入文件。

## SOP

### Step 1 — 获取输入

要求用户提供：
- `index_name`：SIEM 索引名称。
- `backend`：`ELK` 或 `Splunk`。

### Step 2 — 检查现有配置

检查 `DATA/Plugin_SIEM_Indexes/<index_name>.yaml` 是否已存在。
- 如果存在，读取作为基线，后续展示差异。
- 如果不存在，标记为新建。

### Step 3 — 发现字段

调用 `siem_discover_index_fields(index_name=<index_name>, backend=<backend>, max_samples_per_field=20)`。

可选参数：
- `time_range_start` / `time_range_end`：限定采样的时间范围（ISO8601），适用于索引数据量过大或只需采样特定时段的场景。
- `doc_limit`：扫描日志条数，默认 10000。调小可加快返回速度，但样例值可能不全；调大可提高样例覆盖率，但耗时更长。
- `max_samples_per_field`：每个字段返回的最大样例值数量，默认 20。保持默认值即可，后续按自适应规则裁剪。

返回数据包含每个字段的：
- `name`：字段名（嵌套字段用点号路径）
- `type`：后端报告的字段类型
- `sample_values`：样例值列表（保持原始类型，最多 20 条）

**样例值数量自适应规则：**

根据字段特征决定 `sample_values` 的合理数量：

| 字段特征 | 样例数量 | 判断依据 |
|----------|----------|----------|
| 时间戳类（date, timestamp, datetime） | 1 | 类型为 date/timestamp，或值匹配 ISO-8601 / Unix epoch 模式 |
| 高基数长文本（UUID, hash, URL, path, raw log） | 1-2 | 值长度 > 32 字符，或字段名含 uuid/hash/url/path/uri/raw/line |
| 布尔 / 状态开关 | 最多 2 | 类型为 boolean，或值仅为 true/false/0/1 等二元集合 |
| 低基数枚举（≤ 10 个不同值） | 全量 | 去重后值数量 ≤ 10，全部列出 |
| 中基数枚举（11-30 个不同值） | 10 | 取 Top-10 高频值，在描述中注明"共 N 种取值" |
| 高基数枚举（> 30 个不同值） | 5 | 取 Top-5 高频值，在描述中注明"共 N 种取值，仅列出 Top-5" |
| 其他（普通文本、数值） | 5 | 默认 Top-5 |

后端返回的 `sample_values` 数量可能不符合上述规则。处理逻辑：
- 若后端返回数量 **少于** 上表要求：保持原样，不补充。
- 若后端返回数量 **多于** 上表要求：按规则裁剪，并在草案中标注裁剪原因。
- 若后端返回数量 **恰好** 符合：直接采用。

### Step 4 — 生成草案

对每个字段填充完整配置：

| 字段 | 来源 |
|------|------|
| `name` | 直接采用 |
| `type` | 直接采用 |
| `sample_values` | 按上方自适应规则裁剪后采用；若为空列表，在描述中标注"(无样本数据，描述仅供参考)" |
| `description` | 从字段名语义生成，结合 sample_values 补充示例范围；对于枚举类型注明取值数量；sample_values 为空时仅基于字段名推断 |
| `is_key_field` | 按调查价值启发式推断（见下方规则） |

**`is_key_field` 推断规则：**

- `true`：身份、资产、网络四元组、动作、结果、高信号标识字段。
  常见示例：`src_ip`、`dst_ip`、`user`、`username`、`action`、`event_type`、`severity`、`process_name`、`file_hash`、`domain`
- `false`：纯噪音或低价值的元数据字段。
  常见示例：`_id`、`@version`、`beat.hostname`、`log.offset`、`input.type`、`agent.ephemeral_id`

**存在基线时的处理：**

- 保留已有字段的 `description` 和 `is_key_field`（除非该字段类型发生变化）。
- 只对新增字段或类型变化字段重新推断。
- 在草案摘要中明确标注每个字段的来源："保留自基线" 或 "新推断"。

### Step 5 — 展示草案

向用户展示差异摘要和关键字段列表。**不要直接倾倒完整 YAML**——完整 YAML 仅在用户确认写入时生成。

首选回复结构：

**草案摘要**

- 目标索引：`<index_name>`
- 后端：`<backend>`
- 字段总数：`<n>`
- 新增字段：`<list>`（对比基线，如有）
- 类型变化：`<list>`（对比基线，如有）
- `is_key_field=true` 的字段列表

**待确认项**

- `is_key_field` 推断是否合理
- `description` 是否符合命名规范
- 是否需要调整任何字段

### Step 6 — 写入文件

用户确认后，将完整 YAML 写入 `DATA/PLUGINS/SIEM/<index_name>.yaml`。

YAML 顶层结构：

```yaml
name: <index_name>
backend: <backend>
description: <由模型根据 index_name 和字段整体语义自动生成，用户可在 review 时修改>

fields:
  - name: <field_name>
    type: <field_type>
    description: <field_description>
    is_key_field: <true|false>
    sample_values: [<value1>, <value2>, ...]
```

## 澄清规则

- 只有在缺失时才询问 `index_name` 或 `backend`。
- 如果 `siem_discover_index_fields` 返回空字段列表，说明可能 index 名称有误或后端无数据，要求用户确认。
- 如果用户对草案中某些字段的 `is_key_field` 或 `description` 有异议，按用户要求调整后只重新展示受影响的字段，不需要重新展示整个草案。

## 输出规则

- 草案展示时优先用差异摘要 + 关键字段列表，不要直接倾倒完整 YAML。
- 只有在用户确认写入时才生成完整 YAML 文件内容。
- 保持简洁，不要重复解释每个字段的含义。

## 失败处理

- 如果 MCP 工具调用返回连接错误或超时,直接回复失败,提示用户检查 `ASP_MCP_SSE_URL` 环境变量是否已配置,并确认 ASP MCP 服务器已启动.不要尝试重试或绕过.
- 如果 `siem_discover_index_fields` 调用失败，说明错误并要求用户检查 index 名称和后端连通性。
- 如果返回字段数为 0，提示用户确认 index 是否存在且有数据。
- 如果用户要求写入但未经过 review 确认，提醒先完成确认步骤。
