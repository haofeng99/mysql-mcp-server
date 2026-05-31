# 设计规格说明：Issue #106 - 运行时连接注册（add_connection）

## 1. 概述
此功能的目标是允许 LLM 在运行时注册并切换到新的 MySQL/MariaDB 数据库实例，无需重启服务器。这对于动态配置新数据库或多租户探索的环境特别有用。

## 2. 方案：原生 ConnectionManager 集成
我们将实现一个新的 MCP 工具 `add_connection`，利用现有的 `ConnectionManager` 逻辑。该工具将处理 DSN 验证、连通性测试（在添加路径内通过 `Ping`），以及**进程范围内**活动连接的自动切换。

**活动连接范围：** 服务器每个进程使用一个**单一的** `ConnectionManager` 实例。`SetActive(name)` 更新 `ConnectionManager.activeConn`，后者决定 `getDB()` / 工具使用的 `*sql.DB` 池。这是**MCP 服务器进程的全局设置**，而非按 MCP 客户端会话、HTTP 请求或 goroutine。共享此服务器的任何客户端或工具在成功 `add_connection` 后都会看到相同的"活动"连接。

**并发性：** `ConnectionManager` 使用 `sync.RWMutex`（`mu`）。写入者对于变更操作获取 `Lock`；读取者在 `GetActive`、`GetActiveDB`、`List`、`GetServerType` 等操作中获取 `RLock`。对于 add-if-absent，实现在任何 `Ping()` / 隧道工作之前在 `Lock` 下保留 `pendingAdds` 中的名称，确保并发调用方不会重复进行网络 I/O。

## 3. 数据流（`add_connection`）
1. 用户/LLM 以 `name`、`dsn` 和可选 `description` 调用 `add_connection`。
2. 服务器验证名称唯一：未注册且未在 `pendingAdds` 中为正在进行的 add-if-absent 保留。
3. **安全检查（仅限 DSN）：** 服务器解析 DSN，如果 MySQL 用户为 **`root`**（不区分大小写），则拒绝连接。**此检查不检查 MySQL 权限。** 主机/端口策略（允许/拒绝列表等）默认不在代码中强制执行——运维人员应结合网络策略、防火墙规则和 MCP 客户端访问控制。
4. 服务器为 `name` / `dsn` / `description` 构建 `config.ConnectionConfig`。
5. **`ConnectionManager.AddConnectionIfAbsentWithPoolConfig(ctx, connCfg, cfg)`** 应用 SSL/只读默认值，可能建立 SSH 隧道，打开连接池，并使用从 `ctx` 和 `cfg.PingTimeout` 派生的超时运行 **`PingContext`**。注册前**任何**失败时，实现将**关闭**部分打开的 `*sql.DB`，**关闭**为此尝试启动的任何 SSH 隧道，且**不**插入 `ConnectionConfig` 或注册连接池。
6. 如果连接池健康，**`SetActive(name)`** 将进程范围内的活动连接设置为新名称。
7. **回滚：** 如果成功添加后 **`SetActive(name)`** 失败，实现**不得**留下完全注册的连接池而没有一致的活动选择。应调用 `RemoveConnection(name)` 清理，然后向调用方返回错误。如果回滚似乎失败，`toolAddConnection` 返回一个包装了激活失败和回滚失败的单一 Go 错误。
8. 完全成功后，服务器返回命名活动连接的响应。

**替换（`AddConnectionWithPoolConfig`）：** 替换现有名称时，**旧**池和隧道保持注册状态，直到**新** DSN 完全验证。`tearDownNamedConnection(name)` 仅在成功后的**最后**锁定部分运行。

## 4. 结构体定义
```go
type AddConnectionInput struct {
	Name        string `json:"name" jsonschema:"新连接的唯一名称"`
	DSN         string `json:"dsn" jsonschema:"MySQL DSN（user:pass@tcp(host:port)/db）"`
	Description string `json:"description,omitempty" jsonschema:"连接的可选描述"`
}

type AddConnectionOutput struct {
	Success bool   `json:"success" jsonschema:"连接是否添加并激活成功"`
	Active  string `json:"active" jsonschema:"当前活动连接的名称"`
	Message string `json:"message" jsonschema:"状态消息"`
}
```

## 5. 实现细节
- **cmd/mysql-mcp-server/types.go**：添加 `AddConnectionInput` 和 `AddConnectionOutput`。
- **cmd/mysql-mcp-server/connection.go**：暴露 **`AddConnectionIfAbsentWithPoolConfig`**、**`RemoveConnection(name)`**（回滚），保持 **`SetActive(name)`** / **`Ping`** 行为不变。
- **cmd/mysql-mcp-server/tools.go**：
  - 实现 `toolAddConnection`。
  - 解析 DSN 并拒绝 **`root`** 用户（不区分大小写）。
  - 调用 `cm.AddConnectionIfAbsentWithPoolConfig(ctx, connCfg, cfg)`。
  - 调用 `cm.SetActive(name)`；出错时调用 `cm.RemoveConnection(name)` 然后返回错误。
- **cmd/mysql-mcp-server/tool_wrappers.go**：包装 `toolAddConnection`。
- **cmd/mysql-mcp-server/main.go**：在扩展 + 主动加入环境变量时注册工具。

## 6. 约束与边界情况
- **重复名称**：如果名称存在则返回错误（防止意外覆盖），适用于 `AddConnectionIfAbsentWithPoolConfig`。
- **无效 DSN**：如果 DSN 格式无效或不可达则返回错误。
- **权限保护**：阻止运行时注册 `root` 用户 DSN。
- **Ping / 上下文取消**：失败或取消的 `PingContext` 关闭临时资源并使管理器保持不变；可以安全重试。