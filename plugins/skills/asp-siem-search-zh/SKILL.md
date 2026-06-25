---
name: asp-siem-search-zh
description: '用于在 ASP SIEM 中进行日志调查、事件检索、字段探索和结构化分析，适合从模糊线索到精确验证的调查任务。'
argument-hint: 'explore schema [index] | search <keyword> from <UTC start> to <UTC end> | adaptive query <index_name> <time range> [filters] [aggregations] | execute spl <spl_query> | execute esql <esql_query>'
compatibility: connect to asp mcp server
metadata:
  author: Funnywolf
  version: 0.3.0
  mcp-server: asp
  category: cyber security
  tags: [ SIEM, search, SOC, hunting, investigation ]
  documentation: https://asp.viperrtp.com/
---

# ASP SIEM

当用户要在 ASP 中进行 SIEM 调查时，使用这个 skill。重点是帮助用户根据调查目标选择合适的查询方式，并输出对分析有价值的证据。

## 适用场景

- 用户要在 ASP SIEM 中查找、确认或补充安全事件证据。
- 用户想先了解有哪些索引、字段或可用数据源。
- 用户想根据 IOC、告警上下文或关键词搜索相关日志。
- 用户想在已知范围内做进一步过滤、统计或结构化分析。
- 用户想从 alert、artifact 或 case 继续向 SIEM 侧取证。

## 运行规则

- 如果用户请求已经隐含 SIEM 搜索，不要反问用户要选哪种操作。
- 只收集所选路径缺失的必要输入。
- 当用户不知道正确索引或字段时，使用 `siem_explore_schema`。
- `siem_keyword_search` 适合关键词驱动的搜索，既可用于广范围探索，也可用于已知 index 下的快速检索。
- `siem_adaptive_query` 适合精确字段过滤、受控聚合和稳定复现，能力上更接近一个结构化 SIEM API。
- `siem_execute_spl` 适合用户已有 SPL 查询语句，直接执行并返回结果。
- `siem_execute_esql` 适合用户已有 ES|QL 查询语句，直接执行并返回结果。
- 如果用户给出相对时间窗口，根据对话上下文换算为带时区的 ISO 8601 范围；如果时区不明确就询问。当前没有暴露 MCP 时间辅助工具。
- 优化目标是有用证据，而不是最大原始输出。

## MCP 工具契约

- `siem_explore_schema(target_index=None)`
  - 不传 `target_index` 时列出已注册 SIEM index；传入时返回该 index 的字段元数据。
- `siem_keyword_search(keyword, time_range_start, time_range_end, time_field="@timestamp", index_name=None)`
  - `keyword` 是字符串或字符串列表；列表表示 AND 匹配。
  - `time_range_start` 和 `time_range_end` 必填，必须是带时区的 ISO 8601。
  - `index_name` 可选；不传表示跨已注册 backend 做广泛发现。
- `siem_adaptive_query(index_name, time_range_start, time_range_end, time_field="@timestamp", filters=None, aggregation_fields=None)`
  - `filters` 是精确匹配字段过滤；同一字段的列表值表示 OR。
  - `aggregation_fields` 为空时，后端使用 registry key fields。
- `siem_execute_spl(query, time_range_start, time_range_end, limit=100, time_field="@timestamp", index_name=None)`
  - `query`、`time_range_start`、`time_range_end` 必填。`limit` 范围 1-10000。
- `siem_execute_esql(query, time_range_start, time_range_end, limit=100, time_field="@timestamp", index_name=None)`
  - `query`、`time_range_start`、`time_range_end` 必填。`limit` 范围 1-10000。


## 函数说明

### `siem_keyword_search`

适用场景：

- 用户手里有 IP、域名、用户名、主机名、hash、进程名、邮件地址、报错文本或其他关键词，想直接查有没有相关日志。
- 用户还不确定日志在哪个 index，想先在较大范围或全局范围搜索。
- 用户想像 SIEM 检索界面那样多次调整关键词和时间范围，逐步观察结果分布。
- 用户已经知道 index，但当前只需要基于关键词快速找事件，不需要精确字段过滤。

使用方法：

- 必填参数是 `keyword`、`time_range_start`、`time_range_end`。
- `keyword` 可以是单个字符串，也可以是字符串列表；传列表时表示 AND 匹配。
- `index_name` 可选；不传表示全局搜索，传入时表示只在指定源或 index 中搜索。
- `time_field` 默认使用 `@timestamp`，只有在已知数据源使用其他时间字段时才改。

适合输出和解读：

- 返回的是搜索命中的事件集合，适合观察是否命中、命中分布在哪些 backend 或 index、事件时间是否集中、是否出现新的可疑关键词或字段。
- 它既可以作为调查起点，也可以作为调查中的快速二次搜索工具。
- 它不要求调用前就知道精确字段名，因此更适合模糊搜索和探索。

### `siem_adaptive_query`

适用场景：

- 用户已经知道或基本确认日志所在的 `index_name`。
- 用户已经有较明确的过滤条件，例如某个字段必须等于某值，或某几个字段要做精确匹配。
- 用户想做 top-N、分组统计、字段聚合、受控范围查询。
- 用户需要一个更稳定、可复现、结构化的查询，而不是继续做自由关键词搜索。

使用方法：

- 必填参数是 `index_name`、`time_range_start`、`time_range_end`。
- `filters` 是精确字段过滤条件，键是字段名，值可以是单个字符串或字符串列表。
- `aggregation_fields` 是可选的聚合字段列表；只有用户明确需要统计或分组时才添加。
- `time_field` 默认使用 `@timestamp`，只有在目标数据源不是这个字段时才修改。

适合输出和解读：

- 返回的是结构化查询结果，更适合做稳定验证、字段统计和结论支撑。
- 它更像提供给模型使用的一个 SIEM API，不要求一定来自 `siem_keyword_search` 的下一步，但通常需要更清晰的上下文。
- 当用户已经给出 index、时间范围和大致条件时，可以直接使用，不必强行先走关键词搜索。

### `siem_execute_spl`

适用场景：

- 用户已经写好了 SPL 查询语句，想直接执行。
- 用户需要完全控制查询语法，不希望通过关键词搜索或自适应查询的抽象层。
- 用户从 `siem_explore_schema` 了解了字段结构后，自行编写了 SPL。

使用方法：

- `query` 是必填参数，传入原始 Splunk SPL 查询语句。
- `limit` 默认返回 100 条，可按需调整。
- `time_range_start` / `time_range_end` 必填，必须是带时区的 ISO 8601。
- `time_field` 默认 `@timestamp`。
- `index_name` 可选，仅用于输出标记，SPL 中已指定 index 时可不填。

适合输出和解读：

- 返回的是原始 SPL 查询结果，适合直接分析事件详情、字段值分布和时间模式。
- 结果解读应围绕用户的调查目标，而不是罗列所有字段。

### `siem_execute_esql`

适用场景：

- 用户已经写好了 ES|QL 查询语句，想直接执行。
- 用户需要完全控制 ELK 查询语法，包括 ES|QL 特有的函数和管道操作。

使用方法：

- `query` 是必填参数，传入原始 ELK ES|QL 查询语句。
- `limit` 默认返回 100 条，可按需调整。
- `time_range_start` / `time_range_end` 必填，必须是带时区的 ISO 8601。
- `time_field` 默认 `@timestamp`。
- `index_name` 可选，仅用于输出标记，ES|QL 中已指定 index 时可不填。

适合输出和解读：

- 返回的是原始 ES|QL 查询结果，适合直接分析事件详情、字段值分布和时间模式。
- 结果解读应围绕用户的调查目标，而不是罗列所有字段。

### 如何选择

- 如果用户直接给出 SPL 查询语句，用 `siem_execute_spl`。
- 如果用户直接给出 ES|QL 查询语句，用 `siem_execute_esql`。
- 如果线索主要是”关键词”，优先用 `siem_keyword_search`。
- 如果线索主要是”已知 index + 已知字段条件”，优先用 `siem_adaptive_query`。
- 如果用户目标是找事件、看分布、补线索，优先用 `siem_keyword_search`。
- 如果用户目标是精确过滤、聚合统计、稳定复现，优先用 `siem_adaptive_query`。
- 如果用户没有说明字段名且 index 也不确定，不要强推 `siem_adaptive_query`。
- 如果用户已经说明明确的 index 和 filters，也不要机械地先跑 `siem_keyword_search`。

## 决策流程

1. 如果用户问该用哪个索引、有哪些字段，或 SIEM 源如何组织，使用 `siem_explore_schema`。
2. 如果用户给出相对时间窗口，根据上下文换算为带时区的 ISO 8601 范围；如果时区缺失，先询问。
3. 如果用户直接给出 SPL 查询语句，使用 `siem_execute_spl`。
4. 如果用户直接给出 ES|QL 查询语句，使用 `siem_execute_esql`。

## SOP

### 探索 Schema

1. 如果用户不知道目标源，先调用 `siem_explore_schema()`。
2. 如果用户已经知道索引且想要字段结构，调用 `siem_explore_schema(target_index=<index>)`。
3. 解析返回的 JSON。
4. 总结与调查目标最相关的索引、时间字段候选和高信号字段。
5. 推荐下一步查询路径：先全局关键词搜索，或在已明确位置时直接做自适应查询。

### 使用 `siem_keyword_search`

1. 先提取已知最强关键词。
2. 只有当用户真正要求所有条件都匹配时，才把多个关键词规范化为 AND 集合。
3. 要求 UTC 时间戳以 `Z` 结尾。
4. 如果用户未指定 `index_name`，可先全局搜索；如果用户已明确数据源，也可以直接限定到该 index。
5. 调用 `siem_keyword_search`。
6. 解析每条返回的 JSON 字符串，并按用户目标决定输出重点：
   - 如果用户在找事件，优先展示代表性命中。
   - 如果用户在判断日志落点，优先展示 backend、index 分布和时间分布。
   - 如果用户在继续收敛，优先总结下一轮应补充或移除哪些关键词。
7. 根据结果进行下一轮判断：
  - 命中太多时，先收窄时间范围，再补充一到两个高信号关键词。
  - 命中太少或为空时，先移除一个限制性关键词，或适度扩展时间范围。
  - 命中集中在少数 index 时，下一轮优先指定 `index_name`。
  - 当结果已经足够说明日志位置和关键字段时，切换到 `siem_adaptive_query`。

### 使用 `siem_adaptive_query`

1. 要求 `index_name`、UTC 时间范围，以及至少一个精确过滤条件或明确聚合目标。
2. 把过滤条件规范化为精确字段/值对。
3. 只有当用户想要流行度、top-N 统计或分组范围时，才添加 `aggregation_fields`。
4. 调用 `siem_adaptive_query`。
5. 用分析师语言总结过滤范围、命中情况和任何聚合输出。
6. 如果结果不理想，优先判断是 `filters` 太严、字段名不对、时间范围不对，还是其实应回到 `siem_keyword_search` 补充上下文。

### 使用 `siem_execute_spl`

1. 接收用户提供的 SPL 查询语句。
2. 如果用户未指定 `limit`，默认 100；如果用户需要更多结果，可调大。
3. 要求并传入 `time_range_start` / `time_range_end`。
4. 调用 `siem_execute_spl`。
5. 解析返回的 JSON，按用户的调查目标解读结果。
6. 如果结果不理想，检查：SPL 语法是否正确、时间范围是否合适、`limit` 是否足够。

### 使用 `siem_execute_esql`

1. 接收用户提供的 ES|QL 查询语句。
2. 如果用户未指定 `limit`，默认 100；如果用户需要更多结果，可调大。
3. 要求并传入 `time_range_start` / `time_range_end`。
4. 调用 `siem_execute_esql`。
5. 解析返回的 JSON，按用户的调查目标解读结果。
6. 如果结果不理想，检查：ES|QL 语法是否正确、时间范围是否合适、`limit` 是否足够。

### 优化搜索

首选优化动作：

1. 先用 `siem_keyword_search` 在全局或较大范围内建立感觉，不要过早假设 index。
2. 在添加很多新关键词前，先收窄时间范围。
3. 添加一两个高信号关键词，而不是很多弱关键词。
4. 如果查询为空，移除一个限制性关键词，或适度扩大时间窗口。
5. 当广泛搜索返回太多无关数据时，优先指定 `index_name` 或补充更强信号。
6. 当结果已经显示日志主要集中在某类 index 时，下一轮搜索就应该收敛到这些 index。
7. 当用户已经学到足够字段结构和日志落点时，可切换到 `siem_adaptive_query`；但如果用户仍只是想继续搜事件，也可以继续用 `siem_keyword_search`。
8. SPL/ESQL 查询结果不理想时，优先检查语法错误和时间范围，再调整 `limit` 或查询条件。
9. 持续迭代直到结果质量匹配用户目标。



## 回复策略

始终解释搜索的含义，而不只是它返回了什么。

首选回复结构：

### 搜索概览

- 搜索模式：schema 探索、关键词搜索、自适应查询、SPL 查询或 ES|QL 查询
- 关键词集或精确过滤条件
- 时间范围
- 搜索的索引或 `all`
- 如果使用了聚合字段
- 用一两句话给出整体解释，说明当前搜索是在探索、收敛还是验证


### 证据要点

- 对调查重要的关键字段统计。
- 只有在增加价值时才给出代表性记录。
- 对于 schema 探索，只强调对 hunt 重要的索引和字段。
- 对于关键词搜索，说明这是关键词匹配结果，适合继续搜索、看分布、找线索或定位日志来源。
- 对于自适应查询，说明这是结构化过滤或聚合结果，适合做验证、统计和稳定复现。
- 对于 SPL/ESQL 查询，说明这是原始查询结果，适合直接分析事件详情和字段模式。

### 下一步最佳动作

- 收窄时间范围
- 添加一个更强关键词
- 移除一个限制性关键词
- 根据分布切换到特定 index
- 搜索特定索引
- 用精确过滤切换到自适应查询
- 调整 SPL/ESQL 查询语句
- 把有用 SIEM 结果保存为相关 case、alert 或 artifact 上的 enrichment
- 停止，因为证据已经足够

## 澄清规则

- 如果缺少时间范围，询问时间范围。
- 只有当用户没有提供 UTC 且意图时区不清楚时，才询问时区。
- 除非已经明显收敛，否则不要一开始就强制用户提供 `index_name`。
- 只有当广泛搜索可能浪费、用户已经暗示已知源，或自适应查询是正确工具时，才询问 `index_name`。
- 只有当用户想要自适应查询且 schema 仍不清楚时，才询问精确字段名。
- 如果用户说"看看这个事件周围"，从可用 IOC 和时间框架推导出合理的首次搜索，而不是让他们设计查询。
- 如果用户给出了不完整的 SPL 或 ES|QL 语句，提示补充完整后再执行。

## 输出规则

- 保持简洁。
- 默认不要倾倒每条返回记录。
- 优先展示最相关的记录和统计。
- 当返回多个组时，按 backend 和 index 分组结果。
- 对于 schema 探索，呈现候选列表而不是原始字段清单。
- 对于关键词搜索，优先输出能指导下一轮搜索的分布和线索，而不是机械罗列日志。
- 对于自适应查询，优先输出能支撑结论的过滤命中和聚合结果。
- 对于 SPL/ESQL 查询，关注返回字段结构、异常值和时间分布，而不是罗列所有字段。
- 如果没有找到数据，直接说明并建议最可能有用的调整。

## 失败处理

- 如果 MCP 工具调用返回连接错误或超时，直接回复失败，提示用户检查 `ASP_MCP_URL`、`ASP_MCP_API_KEY`、ASGI `/api/mcp` 是否可访问，以及 API key 是否过期、用户是否被禁用。不要尝试重试或绕过。
- 无效时间格式：要求 UTC ISO8601 带尾部 `Z`。
- 空结果：先扩展时间范围或移除一个关键词；如果过早指定了 index，也考虑回到全局关键词搜索。
- 太多命中：先收窄时间范围，再添加信号，必要时切到更可能的 index。
- 未知索引或字段选择：在猜测前使用 `siem_explore_schema`。
- Backend 或源问题：如果结果指示了，说明哪个 backend 或索引失败。
- SPL/ESQL 语法错误：提示用户检查查询语法，指出可能的错误位置。
