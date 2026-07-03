---
name: asp-siem-search
description: "在 ASP SIEM 中探索 schema、搜索日志、执行自适应查询、SPL、ES|QL 和字段发现。"
argument-hint: "schema | keyword <term> --from <time> --to <time> | adaptive | spl | esql"
compatibility: requires asp-cli >= 0.1.0
metadata:
  author: Funnywolf
  version: 1.0.0
  cli: asp
  category: cyber security
  tags: [SIEM, search, SPL, ESQL, investigation, hunting]
  documentation: https://asp.viperrtp.com/
---

# ASP SIEM Search

当用户要在 ASP 中进行 SIEM 调查时，使用这个 skill。重点是根据调查目标选择合适的查询方式，并输出对分析有价值的证据。

默认使用用户的语言回复；如果用户没有明确语言偏好，使用中文。

## 前置检查

如果 ASP CLI 尚未配置或认证状态不明确，先运行 `/asp-setup`。也可以直接执行：

```bash
asp doctor --output json
```

## 适用场景

- 用户要在 ASP SIEM 中查找、确认或补充安全事件证据。
- 用户想了解有哪些索引、字段或可用数据源。
- 用户想根据 IOC、告警上下文或关键词搜索相关日志。
- 用户想在已知范围内做进一步过滤、统计或结构化分析。
- 用户想直接执行已有 SPL 或 ES|QL。
- 用户想从 alert、artifact 或 case 继续向 SIEM 侧取证。

## 运行规则

- 如果用户请求已经隐含 SIEM 搜索，不要反问用户要选哪种操作。
- 只收集所选路径缺失的必要输入。
- 时间范围必须是带时区的 ISO 8601。用户给相对时间时，根据上下文换算；时区不明确时再询问。
- 当用户不知道正确索引或字段时，先探索 schema。
- 关键词搜索适合模糊 IOC、文本、报错、用户名、主机名、域名等线索。
- 自适应查询适合已知 index 和精确字段过滤，也适合稳定复现和聚合统计。
- SPL / ES|QL 适合用户已有原始查询语句，或明确要求使用底层查询语言。
- 优化目标是有用证据，不是最大原始输出。

## 命令契约

### 探索 schema

列出已注册 SIEM 数据源：

```bash
asp siem schema list --output json
```

查看某个 index 的字段元数据：

```bash
asp siem schema show windows-security --output json
```

### 关键词搜索

```bash
asp siem search keyword "8.8.8.8" --from 2026-07-03T00:00:00Z --to 2026-07-03T01:00:00Z --output json
```

限定 index 和时间字段：

```bash
asp siem search keyword "alice" --from 2026-07-03T00:00:00Z --to 2026-07-03T01:00:00Z --time-field "@timestamp" --index-name windows-security --output json
```

- `keyword` 是字符串；逗号分隔值表示 AND 关键词集合。
- `--from` 和 `--to` 必填，必须带时区。
- 不传 `--index-name` 时表示跨已注册 backend 做广泛发现。

### 自适应查询

```bash
asp siem query adaptive windows-security --from 2026-07-03T00:00:00Z --to 2026-07-03T01:00:00Z --filters-json "{\"user.name\":\"alice\"}" --output json
```

带聚合字段：

```bash
asp siem query adaptive windows-security --from 2026-07-03T00:00:00Z --to 2026-07-03T01:00:00Z --filters-json "{\"event.outcome\":\"failure\"}" --aggregation-fields "user.name,source.ip" --output json
```

- `index_name` 必填。
- `filters` 是精确字段过滤；同一字段的列表值表示 OR。
- `aggregation_fields` 为空时，后端使用 registry key fields。

### SPL 查询

```bash
asp siem query spl "index=main error" --from 2026-07-03T00:00:00Z --to 2026-07-03T01:00:00Z --limit 100 --output json
```

- `query`、`--from`、`--to` 必填。
- `--limit` 范围 1-10000。
- `--index-name` 仅用于输出标记；SPL 中已指定 index 时可不传。

### ES|QL 查询

```bash
asp siem query esql "FROM logs-* | LIMIT 10" --from 2026-07-03T00:00:00Z --to 2026-07-03T01:00:00Z --limit 100 --output json
```

- `query`、`--from`、`--to` 必填。
- `--limit` 范围 1-10000。
- `--index-name` 仅用于输出标记。

### 字段发现

```bash
asp siem fields discover windows-security ELK --from 2026-07-03T00:00:00Z --to 2026-07-03T01:00:00Z --doc-limit 10000 --max-samples-per-field 20 --output json
```

- 用于从实际日志样本中发现字段名、样例值和字段分布。
- 适合编写 SIEM index YAML、规则或自适应查询前确认字段。

## 如何选择查询方式

- 用户直接给出 SPL 查询语句时，用 `asp siem query spl`。
- 用户直接给出 ES|QL 查询语句时，用 `asp siem query esql`。
- 线索主要是关键词、IOC、用户名、主机名、hash、报错文本时，优先用 `asp siem search keyword`。
- 线索主要是“已知 index + 已知字段条件”时，优先用 `asp siem query adaptive`。
- 用户目标是找事件、看分布、补线索时，优先用关键词搜索。
- 用户目标是精确过滤、聚合统计、稳定复现时，优先用自适应查询。
- 用户不知道字段名且 index 也不确定时，不要强推自适应查询。
- 用户已经明确 index 和 filters 时，不要机械地先跑关键词搜索。

## SOP

### 探索 schema

1. 如果用户不知道目标源，先调用 `asp siem schema list --output json`。
2. 如果用户已经知道索引且想要字段结构，调用 `asp siem schema show <index> --output json`。
3. 解析返回的 JSON。
4. 总结与调查目标最相关的索引、时间字段候选和高信号字段。
5. 推荐下一步查询路径：全局关键词搜索，或在已明确位置时直接自适应查询。

### 使用关键词搜索

1. 提取已知最强关键词。
2. 只有当用户真正要求所有条件都匹配时，才把多个关键词规范化为 AND 集合。
3. 要求或换算带时区的 ISO 8601 时间范围；UTC 时间优先使用 `Z` 结尾。
4. 用户未指定 index 时可先全局搜索；用户已明确数据源时可限定 `--index-name`。
5. 调用 `asp siem search keyword`。
6. 按用户目标决定输出重点：
   - 找事件：展示代表性命中。
   - 判断日志落点：展示 backend、index、时间分布。
   - 继续收敛：总结下一轮应补充或移除哪些关键词。
7. 根据结果迭代：
   - 命中太多：先收窄时间范围，再补充一到两个高信号关键词。
   - 命中太少或为空：移除一个限制性关键词，或适度扩大时间范围。
   - 命中集中在少数 index：下一轮优先指定 `--index-name`。
   - 已经知道字段结构和日志落点：可切换到自适应查询。

### 使用自适应查询

1. 确认 `index_name`、时间范围，以及至少一个精确过滤条件或明确聚合目标。
2. 把过滤条件规范化为字段和值。
3. 只有当用户想要流行度、top-N、分组统计或范围界定时，才添加 `--aggregation-fields`。
4. 调用 `asp siem query adaptive`。
5. 用分析师语言总结过滤范围、命中情况和聚合输出。
6. 结果不理想时，判断是 filters 太严、字段名不对、时间范围不对，还是应回到关键词搜索补上下文。

### 使用 SPL 查询

1. 接收用户提供的 SPL 查询语句。
2. 如果用户未指定 `limit`，默认 100；需要更多结果时再调大。
3. 确认并传入 `--from` 和 `--to`。
4. 调用 `asp siem query spl`。
5. 按用户调查目标解读结果。
6. 结果不理想时，检查 SPL 语法、时间范围和 limit。

### 使用 ES|QL 查询

1. 接收用户提供的 ES|QL 查询语句。
2. 如果用户未指定 `limit`，默认 100；需要更多结果时再调大。
3. 确认并传入 `--from` 和 `--to`。
4. 调用 `asp siem query esql`。
5. 按用户调查目标解读结果。
6. 结果不理想时，检查 ES|QL 语法、时间范围和 limit。

### 优化搜索

1. 初始探索优先用关键词搜索建立感觉，不要过早假设 index。
2. 添加很多新关键词前，先收窄时间范围。
3. 添加一两个高信号关键词，而不是很多弱关键词。
4. 查询为空时，移除一个限制性关键词，或适度扩大时间窗口。
5. 广泛搜索返回太多无关数据时，优先指定 index 或补充更强信号。
6. 结果显示日志集中在某类 index 时，下一轮搜索收敛到这些 index。
7. 已经学到足够字段结构和日志落点时，可切换到自适应查询。
8. SPL / ES|QL 查询结果不理想时，优先检查语法错误和时间范围，再调整 limit 或查询条件。
9. 持续迭代直到结果质量匹配用户目标。

## 回复策略

始终解释搜索的含义，而不只是它返回了什么。

首选回复结构：

### 搜索概览

- 搜索模式：schema 探索、关键词搜索、自适应查询、SPL 查询或 ES|QL 查询。
- 关键词集或精确过滤条件。
- 时间范围。
- 搜索的索引或 `all`。
- 聚合字段（如果使用）。
- 一两句话说明当前搜索是在探索、收敛还是验证。

### 证据要点

- 对调查重要的关键字段统计。
- 只有在增加价值时才给出代表性记录。
- schema 探索时只强调对 hunt 重要的索引和字段。
- 关键词搜索时说明这是关键词匹配结果，适合看分布、找线索或定位日志来源。
- 自适应查询时说明这是结构化过滤或聚合结果，适合验证、统计和稳定复现。
- SPL / ES|QL 查询时说明这是原始查询结果，适合直接分析事件详情和字段模式。

### 下一步最佳动作

- 收窄时间范围。
- 添加一个更强关键词。
- 移除一个限制性关键词。
- 根据分布切换到特定 index。
- 用精确过滤切换到自适应查询。
- 调整 SPL / ES|QL 查询语句。
- 把有用 SIEM 结果保存为相关 case、alert 或 artifact 上的 enrichment。
- 停止，因为证据已经足够。

## 澄清规则

- 缺少时间范围时，询问时间范围。
- 用户没有提供 UTC 且意图时区不清楚时，询问时区。
- 除非已经明显收敛，否则不要一开始就强制用户提供 `index_name`。
- 只有当广泛搜索可能浪费、用户已暗示已知源，或自适应查询才是正确工具时，才询问 `index_name`。
- 只有当用户想要自适应查询且 schema 仍不清楚时，才询问精确字段名。
- 用户说“看看这个事件周围”时，从可用 IOC 和时间框架推导合理的首次搜索，而不是让用户设计查询。
- 用户给出不完整 SPL 或 ES|QL 时，提示补充完整后再执行。

## 输出规则

- 保持简洁。
- 默认不要倾倒每条返回记录。
- 优先展示最相关的记录和统计。
- 多个组返回时，按 backend 和 index 分组。
- schema 探索呈现候选列表，不输出原始字段清单。
- 关键词搜索优先输出能指导下一轮搜索的分布和线索。
- 自适应查询优先输出能支撑结论的过滤命中和聚合结果。
- SPL / ES|QL 查询关注返回字段结构、异常值和时间分布。
- 没有找到数据时，直接说明并建议最可能有用的调整。

## 失败处理

- 如果 ASP CLI 返回连接错误或超时，直接回复失败，提示用户运行 `asp doctor --output json`，检查 API URL、API key、网络连通性、用户状态和权限。不要尝试绕过认证。
- 无效时间格式：要求带时区的 ISO 8601；UTC 时间建议使用尾部 `Z`。
- 空结果：先扩展时间范围或移除一个关键词；如果过早指定了 index，也考虑回到全局关键词搜索。
- 太多命中：先收窄时间范围，再添加强信号，必要时切到更可能的 index。
- 未知索引或字段：在猜测前使用 schema 探索。
- backend 或数据源问题：如果结果指出了失败 backend 或 index，明确说明。
- SPL / ES|QL 语法错误：提示用户检查查询语法，并指出可能的错误位置。
