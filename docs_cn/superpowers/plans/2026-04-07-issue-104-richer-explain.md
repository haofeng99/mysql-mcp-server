# 更丰富的 EXPLAIN 结构化输出实施计划

> **针对代理工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 来按任务实施此计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 升级 `explain_query` 工具以执行 `EXPLAIN FORMAT=JSON` 并将其解析为强类型的统一 Go 结构体（`UnifiedExplainPlan`），为 LLM 提供可预测的 JSON。

**架构：** 我们将在 `types.go` 中添加统一结构体。然后更新 `tools_extended.go` 中的 `toolExplainQuery`。它将默认为 "json" 格式。如果格式为 "traditional"，则回退到原始表格。如果为 "json"，则执行 `EXPLAIN FORMAT=JSON`，反序列化原始字符串，并将其映射到新的 `UnifiedExplainPlan` 结构体，该结构体在 `ExplainQueryOutput` 中返回。

**技术栈：** Go，标准库（`encoding/json`、`database/sql`）。

---

### 任务 1：在 types.go 中定义统一结构体

**文件：**
- 修改：`cmd/mysql-mcp-server/types.go`

- [ ] **步骤 1：编写结构体**
在 `types.go` 中添加新的统一结构体。注意：`ExplainQueryOutput` 当前持有 `[]map[string]interface{}`。我们将把它改为 `interface{}`，以同时支持传统格式的数组映射（traditional）和新的统一结构体（json）。

```go
type ExplainQueryOutput struct {
	Plan     interface{} `json:"plan" jsonschema:"查询执行计划（传统格式为映射数组，json 格式为对象）"`
	Warnings []string    `json:"warnings,omitempty" jsonschema:"从执行计划中提取的可操作优化建议"`
}

type UnifiedExplainPlan struct {
	QueryCost  float64     `json:"query_cost,omitempty" jsonschema:"查询的总估算代价"`
	Operations []UnifiedOp `json:"operations" jsonschema:"执行计划中的操作列表"`
}

type UnifiedOp struct {
	TableName         string     `json:"table_name,omitempty" jsonschema:"被访问的表"`
	AccessType        string     `json:"access_type,omitempty" jsonschema:"连接类型（如 ALL、ref、range、index）"`
	PossibleKeys      []string   `json:"possible_keys,omitempty" jsonschema:"可用的索引"`
	Key               string     `json:"key,omitempty" jsonschema:"实际使用的索引"`
	KeyLength         string     `json:"key_length,omitempty" jsonschema:"所选键的长度"`
	RowsExamined      int64      `json:"rows_examined,omitempty" jsonschema:"估算读取的行数"`
	Filtered          float64    `json:"filtered,omitempty" jsonschema:"过滤的行百分比（与 EXPLAIN JSON 的 filtered 键匹配）"`
	CostInfo          OpCostInfo `json:"cost_info,omitempty" jsonschema:"详细代价指标"`
	AttachedCondition string     `json:"attached_condition,omitempty" jsonschema:"表访问期间应用的 WHERE 或 ON 子句"`
	Message           string     `json:"message,omitempty" jsonschema:"附加执行细节（如 Using temporary、Using filesort）"`
}

type OpCostInfo struct {
	ReadCost        float64 `json:"read_cost,omitempty"`
	EvalCost        float64 `json:"eval_cost,omitempty"`
	PrefixCost      float64 `json:"prefix_cost,omitempty"`
	DataReadPerJoin string  `json:"data_read_per_join,omitempty"`
}
```

- [ ] **步骤 2：提交**
```bash
git add cmd/mysql-mcp-server/types.go
git commit -m "feat: 在 types.go 中添加统一 explain 结构体"
```

### 任务 2：实现映射逻辑

**文件：**
- 修改：`cmd/mysql-mcp-server/tools_extended.go`

- [ ] **步骤 1：编写映射函数**
在 `tools_extended.go` 中添加一个辅助函数，将原始 JSON 字符串解析为统一结构体。我们将使用基于通用映射的遍历，以安全处理 MySQL 和 MariaDB，避免复杂的嵌套反序列化错误。

```go
func mapRawExplainToUnified(rawJSON string) (UnifiedExplainPlan, error) {
	var raw map[string]interface{}
	if err := json.Unmarshal([]byte(rawJSON), &raw); err != nil {
		return UnifiedExplainPlan{}, err
	}

	plan := UnifiedExplainPlan{}

	if qb, ok := raw["query_block"].(map[string]interface{}); ok {
		costFromInfo := false
		if ci, ok := qb["cost_info"].(map[string]interface{}); ok {
			if costStr, ok := ci["query_cost"].(string); ok {
				if v, err := strconv.ParseFloat(costStr, 64); err == nil {
					plan.QueryCost = v
					costFromInfo = true
				}
			}
		}
		if !costFromInfo {
			if v, ok := float64FromExplainJSONNumber(qb["cost"]); ok {
				plan.QueryCost = v
			}
		}
		if tables, ok := qb["table"].(map[string]interface{}); ok {
			plan.Operations = append(plan.Operations, extractUnifiedOp(tables))
		} else if tablesList, ok := qb["table"].([]interface{}); ok {
			for _, t := range tablesList {
				if tMap, ok := t.(map[string]interface{}); ok {
					plan.Operations = append(plan.Operations, extractUnifiedOp(tMap))
				}
			}
		} else if nestedOps, ok := qb["nested_loop"].([]interface{}); ok {
			for _, nl := range nestedOps {
				if nlMap, ok := nl.(map[string]interface{}); ok {
					if tMap, ok := tableMapFromNestedLoopStep(nlMap); ok {
						plan.Operations = append(plan.Operations, extractUnifiedOp(tMap))
					}
				}
			}
		}
	}

	return plan, nil
}
// extractUnifiedOp、float64FromExplainJSONNumber、tableMapFromNestedLoopStep：参见 tools_extended.go
```

- [ ] **步骤 2：更新 toolExplainQuery 逻辑**
修改 `toolExplainQuery` 以处理 format 参数。默认使用 JSON。更新 SQL 执行以在需要时注入 `FORMAT=JSON`。

- [ ] **步骤 3：运行构建**
运行：`go build ./...`
预期：PASS

- [ ] **步骤 4：提交**
```bash
git add cmd/mysql-mcp-server/tools_extended.go
git commit -m "feat: 实现统一 explain 逻辑并更新工具格式处理器"
```

### 任务 3：更新单元测试

**文件：**
- 修改：`cmd/mysql-mcp-server/tools_extended_test.go`

- [ ] **步骤 1：为映射逻辑编写测试**
- [ ] **步骤 2：运行单元测试**
- [ ] **步骤 3：更新现有的 TestExplainQuery 传统测试**
- [ ] **步骤 4：运行所有测试** — 运行：`make test`，预期：PASS
- [ ] **步骤 5：提交**