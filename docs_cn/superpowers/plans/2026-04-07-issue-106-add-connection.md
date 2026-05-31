# 运行时连接注册（add_connection）实施计划

> **针对代理工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 来按任务实施此计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 实现 `add_connection` 工具，允许在运行时注册并切换到新的 MySQL 连接。

**架构：** 我们将在 `types.go` 中定义输入/输出类型，在 `tools.go` 中实现工具逻辑（包括对 root 的安全性检查），使用自定义闭包包装工具以访问全局配置和连接管理器，并在 `main.go` 中注册它。

**技术栈：** Go，标准库，`github.com/go-sql-driver/mysql`，`github.com/modelcontextprotocol/go-sdk/mcp`。

---

### 任务 1：在 types.go 中定义结构体

**文件：**
- 修改：`cmd/mysql-mcp-server/types.go`

- [ ] **步骤 1：编写结构体**
将 `AddConnectionInput` 和 `AddConnectionOutput` 添加到 `types.go` 中。

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

- [ ] **步骤 2：提交**
```bash
git add cmd/mysql-mcp-server/types.go
git commit -m "feat: 在 types.go 中添加 AddConnectionInput/Output 结构体"
```

---

### 任务 2：在 tools.go 中实现工具逻辑

**文件：**
- 修改：`cmd/mysql-mcp-server/tools.go`

- [ ] **步骤 1：实现 toolAddConnection**
添加工具的核心逻辑。此函数需要访问 `ConnectionManager` 和 `config.Config`。由于大多数工具使用 `wrapTool` 的特定签名，我们将定义此函数并在后续处理依赖注入。

```go
func toolAddConnection(
	ctx context.Context,
	req *mcp.CallToolRequest,
	input AddConnectionInput,
	cm *ConnectionManager,
	cfg *config.Config,
) (*mcp.CallToolResult, AddConnectionOutput, error) {
	name := strings.TrimSpace(input.Name)
	dsn := strings.TrimSpace(input.DSN)
	if name == "" || dsn == "" {
		return nil, AddConnectionOutput{}, fmt.Errorf("名称和 DSN 为必填项")
	}

	// 1. 检查名称是否已存在
	conns := cm.List()
	for _, c := range conns {
		if c.Name == name {
			return nil, AddConnectionOutput{}, fmt.Errorf("连接 '%s' 已存在", name)
		}
	}

	// 2. 安全检查：拒绝 root 用户
	mysqlCfg, err := mysql.ParseDSN(dsn)
	if err != nil {
		return nil, AddConnectionOutput{}, fmt.Errorf("无效的 DSN: %w", err)
	}
	if mysqlCfg.User == "root" {
		return nil, AddConnectionOutput{}, fmt.Errorf("安全策略：不允许在运行时注册 'root' 用户")
	}

	// 3. 添加连接
	connCfg := config.ConnectionConfig{
		Name:        name,
		DSN:         dsn,
		Description: input.Description,
	}
	if err := cm.AddConnectionWithPoolConfig(connCfg, cfg); err != nil {
		return nil, AddConnectionOutput{}, fmt.Errorf("添加连接失败: %w", err)
	}

	// 4. 自动切换到该连接
	if err := cm.SetActive(name); err != nil {
		return nil, AddConnectionOutput{}, fmt.Errorf("激活连接失败: %w", err)
	}

	return mcp.NewToolResultText(fmt.Sprintf("成功添加并切换到连接 '%s'。", name)),
		AddConnectionOutput{
			Success: true,
			Active:  name,
			Message: fmt.Sprintf("已添加并切换到连接 '%s'", name),
		}, nil
}
```

- [ ] **步骤 2：提交**
```bash
git add cmd/mysql-mcp-server/tools.go
git commit -m "feat: 实现 toolAddConnection 逻辑"
```

---

### 任务 3：包装并注册工具

**文件：**
- 修改：`cmd/mysql-mcp-server/tool_wrappers.go`
- 修改：`cmd/mysql-mcp-server/main.go`

- [ ] **步骤 1：在 tool_wrappers.go 中创建专用包装器**
大多数工具使用 `wrapTool` 包装。`toolAddConnection` 需要额外参数。我们将创建一个手动包装器或扩展 `tool_wrappers.go`。

```go
// 添加到 tool_wrappers.go
func wrapAddConnection(cm *ConnectionManager, cfg *config.Config) func(context.Context, *mcp.CallToolRequest, AddConnectionInput) (*mcp.CallToolResult, AddConnectionOutput, error) {
	return func(ctx context.Context, req *mcp.CallToolRequest, input AddConnectionInput) (*mcp.CallToolResult, AddConnectionOutput, error) {
		return toolAddConnection(ctx, req, input, cm, cfg)
	}
}
```

- [ ] **步骤 2：在 main.go 中注册工具**
在 `main.go` 中定位工具注册块并添加 `add_connection`。

```go
	// 在 main.go 中其他工具注册之后
	mcp.AddTool(server, &mcp.Tool{
		Name:        "add_connection",
		Description: "在运行时注册并切换到新的 MySQL 连接。",
	}, wrapTool("add_connection", wrapAddConnection(connManager, cfg)))
```

- [ ] **步骤 3：运行构建** — 运行：`go build ./...`，预期：PASS
- [ ] **步骤 4：提交**

---

### 任务 4：通过单元测试验证

**文件：**
- 创建：`cmd/mysql-mcp-server/connection_tool_test.go`

- [ ] **步骤 1：编写测试用例** — 测试 `root` 被拒绝，且有效连接（模拟）被添加和激活。
- [ ] **步骤 2：运行测试** — 运行：`go test -v ./cmd/mysql-mcp-server/...`，预期：PASS
- [ ] **步骤 3：提交**