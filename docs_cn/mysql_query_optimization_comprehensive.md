# 综合 MySQL 查询优化指南

本指南为高级开发者和数据库管理员提供 MySQL 查询优化的深度技术洞察和运维最佳实践。它建立在基础知识之上，涵盖 MySQL 优化器的底层工作原理、高级索引策略和运维维护。

---

## 1. 优化器统计信息：基数与选择性

理解 MySQL 查询优化器如何做决策对于编写高效查询至关重要。优化器在很大程度上依赖表数据的统计信息。

### 选择性
选择性是指不同值的数量与表中总行数的比值，范围从 0 到 1。
- **高选择性（接近 1）：** 唯一值（例如主键、邮箱地址）。这些列上的索引非常有效。
- **低选择性（接近 0）：** 大量重复值（例如状态、性别、布尔标志）。优化器通常会忽略这些列上的索引，因为全表扫描可能比先读取索引再查找行更快。

### 基数
基数是指索引中估计的唯一值数量。
- 你可以使用 `SHOW INDEX FROM table_name;` 来查看。
- 查询优化器使用基数来决定是否使用索引以及以什么顺序连接表。
- **注意：** `SHOW INDEX` 中的基数值是一个*估计值*。如果它变得严重不准确，优化器可能会选择糟糕的执行计划。运行 `ANALYZE TABLE table_name;` 可以更新这些统计信息。

---

## 2. 高级索引策略

除了基本的单列和复合索引之外，MySQL 还支持多种高级索引功能，可以显著提高特定查询模式的性能。

### 函数索引（MySQL 8.0+）
在 MySQL 8.0 之前，在 `WHERE` 子句中对索引列应用函数（例如 `WHERE YEAR(created_at) = 2023`）会使索引无效。函数索引通过对表达式的*结果*建立索引来解决这个问题。

```sql
-- 创建函数索引
CREATE INDEX idx_created_year ON orders ((YEAR(created_at)));

-- 此查询现在将使用 idx_created_year 索引
SELECT * FROM orders WHERE YEAR(created_at) = 2023;
```

### 降序索引（MySQL 8.0+）
虽然 MySQL 一直支持反向扫描索引，但降序索引将数据按降序存储，消除了反向索引扫描的代价。这对于具有混合方向排序的多列查询特别有用。

```sql
-- 创建混合排序的复合索引
CREATE INDEX idx_score_date ON posts (score DESC, created_at ASC);

-- 此查询被上述索引完美优化
SELECT * FROM posts ORDER BY score DESC, created_at ASC LIMIT 10;
```

### 前缀索引
对于非常长的字符串列（如 `VARCHAR(255)` 或 `TEXT`），索引整个字符串会浪费空间并降低缓存效率。前缀索引允许你只索引前 *N* 个字符。

```sql
-- 仅索引 URL 的前 20 个字符
CREATE INDEX idx_url_prefix ON web_pages (url(20));
```

*提示：* 要找到合适的前缀长度，可以将不同前缀的数量与不同值的总数进行比较。你希望一个长度能捕获大部分唯一性而不浪费空间。

---

## 3. 深入查询计划分析

`EXPLAIN` 命令是你理解 MySQL 如何执行查询的主要工具。MySQL 8.0 引入了更可读、更详细的输出格式。

### EXPLAIN ANALYZE
`EXPLAIN ANALYZE`（在 MySQL 8.0.18 中引入）实际执行查询，并同时提供*实际*执行时间、行数以及优化器的*估计值*。这对于识别优化器做出错误假设的地方非常有价值。

```sql
EXPLAIN ANALYZE SELECT * FROM users u JOIN orders o ON u.id = o.user_id WHERE u.status = 'active';
```

### FORMAT=TREE
`TREE` 格式提供了执行计划的层级视图，展示了从下往上的数据流。它使复杂的 JOIN 和子查询更容易追踪。

```sql
EXPLAIN FORMAT=TREE SELECT * FROM ...
```

**需要关注的关键元素：**
- **Table scan on <table_name>：** 表示全表扫描。
- **Index lookup on <index_name>：** 表示高效的索引查找。
- **Filter：** 表示行在*读取后*被丢弃。读取行和返回行之间的差异很大表明缺少索引。
- **Temporary table：** 表示 MySQL 必须将中间结果写入内存或磁盘。通常由复杂的 `GROUP BY` 或 `ORDER BY` 操作引起。

---

## 4. 高级执行技术

MySQL 实现了多种内部优化策略来加速复杂查询。理解这些有助于你设计模式和查询以触发它们。

### 索引条件下推（ICP）
通常，MySQL 使用索引检索行，然后评估 `WHERE` 子句的其余部分。使用 ICP，如果 `WHERE` 子句的部分条件可以使用*索引中已经存在的列*进行评估，MySQL 会在读取完整行*之前*在存储引擎层过滤它们。
- **收益：** 在读取本将被丢弃的行时大幅减少 I/O。
- **在 EXPLAIN 中查看：** `Extra` 列中的 `Using index condition`。

### 多范围读取（MRR）
当使用二级索引读取行时，主键查找可能导致随机磁盘 I/O。MRR 扫描二级索引，收集主键，对它们排序，然后按主键顺序获取行（顺序 I/O）。
- **收益：** 对机械硬盘有巨大的性能提升，即使在 SSD 上也有显著的 CPU/缓存收益。
- **在 EXPLAIN 中查看：** `Extra` 列中的 `Using MRR`。

### Skip Scan 优化（MySQL 8.0.13+）
传统上，如果你在 `(A, B)` 上有索引，但查询只过滤 `B`，则无法使用该索引。如果列 `A` 具有低基数，Skip Scan 改变了这一点。MySQL 将"跳过"`A` 的不同值，并对每个值执行 `B` 的索引查找。
- **收益：** 当 `(A, B)` 已存在且 `A` 只有少量唯一值时，避免了对冗余索引如 `(B)` 的需求。
- **在 EXPLAIN 中查看：** `Extra` 列中的 `Using index for skip scan`。

---

## 5. 性能运维指南

查询优化不是在真空中进行的。服务器环境和持续监控同样重要。

### 慢查询日志分析
慢查询日志是在生产环境中查找问题查询的权威来源。
- **启用：** 设置 `slow_query_log = 1` 和 `long_query_time = 1`（甚至 0.1 秒）。
- **工具：** 使用 `mysqldumpslow` 或 `pt-query-digest`（Percona 工具包）来聚合和分析日志。重点关注具有高*总*执行时间的查询，而不仅仅是单个最慢的查询。

### Performance Schema 监控
Performance Schema 是 MySQL 内置的低级仪表引擎。
- 它可以跟踪等待事件、锁争用、内存使用和查询执行阶段。
- **Sys Schema：** MySQL 5.7+ 包含 `sys` schema，提供了 Performance Schema 之上的人类可读视图。
- 示例：查找等待锁的查询：`SELECT * FROM sys.innodb_lock_waits;`

### 缓冲池调优
InnoDB 缓冲池是 MySQL 在内存中缓存数据和索引的地方。它是性能方面最关键的配置参数。
- **经验法则：** 在专用数据库服务器上，将 `innodb_buffer_pool_size` 设置为总物理 RAM 的 60-80%。
- 缓冲池太小会导致过多的磁盘 I/O（颠簸）。
- 监控 `Innodb_buffer_pool_wait_free`；如果它 > 0，则你的缓冲池正处于压力之下。

---

## 6. 维护检查清单

持续的维护可确保查询性能不会随着数据增长而随时间下降。

### 每日
- [ ] 查看慢查询日志摘要（例如通过每日 `pt-query-digest` 报告）。
- [ ] 监控 CPU、磁盘 I/O 和内存使用警报。
- [ ] 检查长时间运行的事务或死锁。

### 每周
- [ ] **分析表：** 对具有大量 `INSERT/UPDATE/DELETE` 变更的表运行 `ANALYZE TABLE`，以刷新优化器统计信息。（可以在非高峰时段自动化或安排任务）。
- [ ] 查看前 10 个最耗费资源的查询，评估是否需要新索引。
- [ ] 使用 `sys.schema_unused_indexes` 和 `sys.schema_redundant_indexes` 检查未使用或重复的索引。

### 每月
- [ ] **容量规划：** 查看数据增长趋势，预测何时将触及存储或内存限制。
- [ ] **数据归档：** 识别很少查询的大型旧数据，将其归档到历史表，以保持活跃表小而快。
- [ ] 查看查询模式，了解应用行为的结构性变化。