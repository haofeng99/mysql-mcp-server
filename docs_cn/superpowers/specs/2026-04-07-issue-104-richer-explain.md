# 设计规格说明：Issue #104 - 更丰富的 EXPLAIN 结构化输出

## 1. 概述
此功能的目标是升级 `explain_query` 工具，为查询执行计划提供强类型、统一的 JSON 模式。这确保 LLM 接收到可预测的结构化数据用于查询分析，无论后端引擎是 MySQL（8.x、9.x）还是 MariaDB（10.x、11.x）。

## 2. 方案：强类型统一模式
我们将实现一个规范化层，执行 `EXPLAIN FORMAT=JSON`，反序列化引擎特定的 JSON，并将其映射到统一的 Go 结构体。LLM 将在统一的 JSON 输出上执行所有分析；**自动生成的警告仅用于传统的 `traditional` 格式路径**（参见 §3 和 §5）。

## 3. 数据流
1. 用户通过 MCP 请求 `explain_query`。
2. **格式和 SQL 发送到服务器**
   - 如果 `format` 字段**被省略或为空**，服务器将其视为 **`"json"`**：运行 **`EXPLAIN FORMAT=JSON <query>`** 并返回结构化输出（参见步骤 4-6）。
   - 如果 **`format` 明确为 `"traditional"`**，服务器运行普通的 **`EXPLAIN <query>`**（不带 `FORMAT=JSON`），并以行映射数组的形式返回**之前的原始表格计划**（与结构化 JSON 之前的行为相同）。优化**警告**仅在此路径中附加，当 `format` 为 `"traditional"` 时（与 §2 一致：默认 JSON/统一路径不生成自动警告）。
   - 其他显式值（例如 **`"tree"`**）遵循非 JSON 路径：`EXPLAIN` 以所选格式运行，行作为扫描的列映射返回（而不是统一结构体）。
3. 当有效格式为 **`json`** 时，服务器执行 **`EXPLAIN FORMAT=JSON <query>`**。
4. 服务器反序列化数据库返回的原始 JSON 字符串（单行/单列）。
5. 映射函数遍历原始 JSON 并填充 **`UnifiedExplainPlan`** 结构体。
6. 结构化的 **`ExplainQueryOutput`** 返回给客户端（参见 §5 了解错误处理和 **`Plan`** 类型）。

### 3.1 `Operations []UnifiedOp`：嵌套的表示方式
统一模式使用**扁平列表**，而非嵌套的 `UnifiedOp` 值树。

- **`UnifiedExplainPlan.Operations`** 是一个按**执行顺序**排列的 **`UnifiedOp`** 条目的**有序列表**，由深度优先遍历引擎 JSON 生成：首先是驱动表访问，然后是按顺序的每个连接表/嵌套循环步骤。
- **`UnifiedOp` 不**携带 **`SubOperations`**、**`Children`** 或父指针。原始计划中的层级关系（例如 `nested_loop` 数组、子查询、派生表）被**扁平化**为此序列；消费者仅通过位置和原始 JSON 中的重复模式（如需要）推断结构。
- **编码摘要：** MySQL 或 MariaDB JSON 中作为嵌套 `query_block` 或深层 `nested_loop` 树出现的子查询/派生表/多级嵌套，通过按顺序发出该遍历中遇到的每个叶节点 `table` 对象的一个 **`UnifiedOp`** 来映射。规范不需要为子树根提供单独的字段。

### 3.2 将复杂查询扁平化到 `Operations` 中（示意）
引擎的 JSON 比简单的表扫描序列更丰富；**统一**模型仍然存储**单个有序的 `[]UnifiedOp`**。以下 "JSON 草图" 是概念性的——实际计划因版本和优化器而异。

**(1) 带子查询的 WHERE** — 例如 `SELECT * FROM orders o WHERE o.user_id = (SELECT id FROM users WHERE email = ?)`

- **JSON 草图：** 外部 `query_block` 可能列出 `orders` 的 `table`，以及一个用于 `users` 查找的嵌套 `query_block`（或 `materialized_from_subquery` / `subqueries` 区域，取决于引擎）。深度优先顺序中的每个可解析 **`table`** 叶节点成为一个 **`UnifiedOp`**。
- **结果 `Operations`（顺序）：** `[ UnifiedOp{"TableName": "orders", …}, UnifiedOp{"TableName": "users", …} ]` — 外部驱动表在前，然后是内部/子查询表访问。

**(2) FROM 中的派生表** — 例如 `SELECT * FROM (SELECT id FROM t1) AS d JOIN t2 ON …`

- **JSON 草图：** 优化器可能显示 `nested_loop`，第一步扫描 `t1`（在派生表 `d` 内），第二步扫描 `t2`，或一个临时/物化节点后跟连接；按顺序访问的每个 **`table`** 对象生成一个 **`UnifiedOp`**。
- **结果 `Operations`（顺序）：** `[ UnifiedOp{"TableName": "t1" …}, UnifiedOp{"TableName": "t2" …} ]`（名称示意；某些引擎中派生名称可能出现为 `<subqueryN>`）。

**(3) 物化子查询 / 临时表** — 例如 `WHERE x IN (SELECT …)` 带物化

- **JSON 草图：** 计划可能包含显式物化或 block-nl-join 节点；尽管如此，遍历结构下的每个 **`table`** 叶节点按照访问顺序成为 **`UnifiedOp`**。
- **结果 `Operations`（顺序）：** **`UnifiedOp`** 条目的有序列表，与 **`table`** 对象的序列匹配——物化不增加单独的统一字段；它可能仅出现在原始 JSON 或 **`message`** 中。

## 4. 结构体定义
与 `cmd/mysql-mcp-server/types.go` 中的 **`UnifiedExplainPlan`** / **`UnifiedOp`** 对齐。

### 4.1 `ExplainQueryOutput`（当前实现）
MCP 工具返回 **`ExplainQueryOutput`**：
- **`Plan`** — **`interface{}`**（JSON 多态），而非单独的顶层 `unified_plan` / `raw_json` 字段。调用方通过类型区分形状。
- **`format` 为 `json`（默认）：** **`Plan`** 在规范化成功时通常为 **`UnifiedExplainPlan`**；如果规范化失败但服务器返回了 JSON 字符串，**`Plan`** 为**原始 JSON 字符串**（回退）。
- **`format` 为 `traditional`：** **`Plan`** 为 **`[]map[string]interface{}`**（表格行）。
- **其他格式（如 tree）：** **`Plan`** 为列扫描产生的 **`[]map[string]interface{}`**。
- **`Warnings`** — 仅在 **`format=traditional`** 时填充（来自表格分析的优化提示）；对于 JSON/统一输出则省略或为空。

## 5. 错误处理（已实现）
此章节与 `tools_extended.go` 中的 **`toolExplainQuery`** 和 **`mapRawExplainToUnified`** 匹配。

- **EXPLAIN 失败**（语法、权限、连接、JSON 空结果集等）：处理器返回**非 nil 的 Go `error`**，且不返回成功结构化的结果。在这些路径上 **`ExplainQueryOutput`** 为零值。
- **`EXPLAIN FORMAT=JSON` 成功**并返回 JSON 字符串：服务器反序列化一次。**`mapRawExplainToUnified`**：
  - 如果计划字符串的 **JSON 反序列化**失败 → 返回 **`error`**；**`toolExplainQuery`** 然后将 **`Plan`** 设置为**原始字符串**，以便客户端仍能收到有效负载。
  - 如果反序列化成功但**规范化失败**，因为 **`filtered`** 字段违反 §7.6 规则 4 → **`toolExplainQuery`** 返回**非 nil 错误**（描述性消息）；成功路径上无 **`Plan`** 负载。
  - 如果反序列化成功且映射完成，则生成 **`UnifiedExplainPlan`**（如果文档缺少预期键，**`Operations`** 可能为空）。

## 6. 实现细节
- **cmd/mysql-mcp-server/types.go**：定义 **`UnifiedExplainPlan`**、**`UnifiedOp`**、**`OpCostInfo`**、**`ExplainQueryOutput`**。
- **cmd/mysql-mcp-server/tools_extended.go**：
  - **`toolExplainQuery`**：未设置时默认 **`format`** 为 **`json`**；**`traditional`** 选择表格 **`EXPLAIN`**。
  - JSON 路径使用 **`EXPLAIN FORMAT=JSON`**；无行数/扫描/**`rows.Err()`** 时出错。
  - 解析来自服务器的 JSON 字符串（单行/单列）。
  - **`mapRawExplainToUnified`** 将 **`query_block`** 下的引擎 JSON 映射到 **`UnifiedExplainPlan`**。
  - 返回 **`ExplainQueryOutput`**，**`Plan`** 设置为 **`UnifiedExplainPlan`**，或仅当计划字符串不是有效顶层 JSON 时为原始 JSON 字符串。

## 7. MySQL vs MariaDB JSON：嵌套形状和映射
MySQL 和 MariaDB 都暴露顶层 **`query_block`**，但在**代价位置**、**连接嵌套**和 **MariaDB 特定的**连接节点（例如 **`block-nl-join`**）方面有所不同。

### 7.1 示例：MySQL — `cost_info` + 带 `table` 对象的 `nested_loop`
MySQL 通常在 `query_block.cost_info.query_cost`（字符串）中放置总代价。每个连接步骤为 **`nested_loop[]` → `table`**。

### 7.2 示例：MariaDB — 块级 `cost`、单个 `table`、数值型 `filtered`
MariaDB 可能暴露 `query_block.cost`（数字）而非 `cost_info.query_cost`。简单计划直接使用 `query_block.table`（不总是 `nested_loop`）。

### 7.3 示例：MariaDB — 带 `block-nl-join` 包装器的 `nested_loop`
MariaDB 可能在 `block-nl-join` 中包装连接步骤；**`table`** 映射则在 `block-nl-join.table` 下，而非同一对象上的同级 **`table`** 键。

### 7.4 映射表 → `UnifiedExplainPlan` / `UnifiedOp`

| 目标字段 | MySQL 源路径 | MariaDB 源路径 |
|--------------|----------------------|-------------------------|
| **`UnifiedExplainPlan.QueryCost`** | `query_block.cost_info.query_cost`（解析字符串为 float） | 优先 `query_block.cost_info.query_cost`（如存在）；否则 `query_block.cost`（数值） |
| **`UnifiedExplainPlan.Operations`** | 从 `query_block.table`/`query_block.nested_loop[*].table` 收集 | 同 MySQL，外加 `query_block.nested_loop[*].block-nl-join.table` |
| **`UnifiedOp.TableName`** | `table.table_name` | `table.table_name` |
| **`UnifiedOp.AccessType`** | `table.access_type` | `table.access_type` |
| **`UnifiedOp.RowsExamined`** | `table.rows_examined_per_scan` 或 `table.rows` | `table.rows` 或 `table.rows_examined_per_scan` |
| **`UnifiedOp.Filtered`** | `table.filtered`（参见 §7.6 规则 4） | 同上 |
| **`UnifiedOp.AttachedCondition`** | `table.attached_condition` | `table.attached_condition`，对于 `block-nl-join` 进行**包装器合并**（§7.5） |

### 7.5 `block-nl-join` 和 `attached_condition`（确定性）
遍历 `query_block.nested_loop` 时（或解析包含 `block-nl-join` 的步骤时）：
1. 从 `block-nl-join.table` 获取内部表映射。
2. **必须**将 `block-nl-join.attached_condition` 合并到该表映射中，**仅当**表对象**尚未**定义 `attached_condition`（非空键在 `table` 对象上优先）。
3. 将结果映射传递给 `extractUnifiedOp`。这是**非可选**的。

### 7.6 回退规则（确定性）
1. **查询级代价：** 优先使用 `query_block.cost_info.query_cost`；否则 `query_block.cost`（MariaDB）；否则不设置 `QueryCost`（零/JSON 中省略）。
2. **Operations 列表：** 优先 `query_block.table`（标量或数组）；否则按顺序遍历 `query_block.nested_loop`。
3. **每操作行数：** 优先 `rows_examined_per_scan`，然后 `rows`。
4. **每操作 filtered：** 键存在且值不为 `null` 时，必须解析为数值或数值字符串；否则视为规范化错误，拒绝请求。
5. **仅在一个引擎上存在的字段：** 路径存在时映射；否则省略/零值。

## 8. 向后兼容性
**`ExplainQueryInput`** 保留 `format` 字段。
- **`format` 省略** → 视为 `json`（解析成功时为结构化 `UnifiedExplainPlan`）。
- **`format: "traditional"`** → 传统 `EXPLAIN` 表格行和警告分析，保持不变。
- 显式 **`json`** → 与默认 JSON 路径相同。