# 与 Oracle MCP Server 对比及建议更新

本报告对比当前基于 Go 的 `mysql-mcp-server` 与 Oracle 官方版本（`oracle/mcp`），并提出潜在改进建议。

## 功能对比

| 功能领域 | 本地版本 (`askdba/mysql-mcp-server`) | Oracle 版本 (`oracle/mcp`) |
| :--- | :--- | :--- |
| **主要目标** | 通用用途、性能、安全。 | MySQL AI、HeatWave、OCI 用户。 |
| **数据库工具** | 深度内省（20+ 工具）。 | 仅基础 SQL 执行。 |
| **AI / 向量** | 原生 MySQL 9.0 向量支持。 | 原生 HeatWave GenAI 和 ML 工具。 |
| **连接性** | 标准 DSN、SSL/TLS。 | 通过堡垒主机进行 SSH 隧道连接。 |
| **自然语言** | LLM 驱动（通过 MCP 客户端）。 | `ask_nl_sql`：内置自然语言转 SQL。 |
| **云集成** | 无。 | OCI 对象、区间、存储桶。 |
| **可观测性** | Token 跟踪、审计日志。 | 无。 |
| **架构** | 单一 Go 二进制文件（零依赖）。 | Python（需要环境配置）。 |

## 对 `askdba/mysql-mcp-server` 的建议更新

### 1. SSH 隧道（堡垒主机）
支持通过 SSH 堡垒主机连接 MySQL 实例。这是生产数据库的常见需求。
- **复杂度**：高
- **优先级**：高

### 2. 自然语言转 SQL（`ask_nl_sql`）
添加一个专门处理 NL 到 SQL 转换的工具，使用模式上下文。虽然 Claude 原生支持此功能，但服务端工具可以提供更"接地"的提示，包含精确的表模式。
- **复杂度**：中
- **优先级**：中

### 3. 连接特定的执行
修改核心工具以接受可选的 `connection` 参数。目前，服务器使用"切换-运行"模型。允许 `run_query(sql, connection="prod")` 将提高多数据库环境的可用性。
- **复杂度**：中
- **优先级**：中

### 4. HeatWave / GenAI 封装
如果连接的服务器支持，添加对 MySQL HeatWave 特定函数（如 `ml_generate`）的支持。可通过 `MYSQL_MCP_HEATWAVE=1` 标志来控制。
- **复杂度**：低
- **优先级**：低

### 5. 数据导入（本地/云端）
用于协助加载向量存储数据的工具，类似于 `load_vector_store_local`。这可以促进 RAG 能力数据库的初始设置。
- **复杂度**：中
- **优先级**：低

## 下一步
1. 调研和对比
2. 获取优先级反馈
3. 为选定功能创建独立的 issue