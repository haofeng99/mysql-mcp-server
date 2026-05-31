# 开放待办 — PR 分组与"下一个 PR"推荐

> **针对代理工作者：** 这是一个**规划**文档（issue 分组 + 排序），而非带有复选框的实现计划。要实现选定的部分，请衍生一个后续计划，使用 `writing-plans` / `subagent-driven-development`，遵循 **AGENTS.md** 和 **workflow.md**。

**目标：** 为**范围内的**开放 issue（**#103、#104、#106**）定义**可审查的 PR 边界**，与 **workflow.md** 保持一致（"当多个 issues 共享一个实现时合并为一个内聚的 PR；当相互独立时分别创建 PR"）。

**开放 issue 检查清单（vs GitHub）：** 当前所有**开放**的 issue 要么在下表中（**#103–#106**），要么列为**超出范围**（**#24、#64、#80**）。没有遗漏。如需包含推迟项目的表格快照，请参见 **[workflow.md](../../../workflow.md)** → *开放待办快照*。

**来源：** 并行 **explore** 子代理（独立代码库映射）+ `gh issue view` 正文 + **AGENTS.md** / **workflow.md**。

**技术栈：** Go 1.24+，现有的 `cmd/mysql-mcp-server`、`internal/config`、`internal/util`，扩展工具在 `tools_extended.go` 中。

**本文档范围外（以后再看）：** **#24**（TiDB）、**#64** / **#80**（本地 LLMs 和 `ask_nl_sql`，包括任何组合的服务端 NL→SQL 轨道）。

---

## 子代理综合分析（仅范围内 issues）

| Issue | 主题 | 与其他 issues 的重叠 |
|-------|--------|---------------------|
| **#104** | `explain_query` 结构化输出 + 文档 | 隔离于扩展工具 + README；与写入或连接**无**依赖 |
| **#103** | `write_query` + 确认 + 审计 | 核心 SQL 路径、验证器、安全测试；与 #106 **不同**子系统 |
| **#106** | `add_connection` / HTTP 持久化 | `ConnectionManager`、可选 HTTP；与 #103 **不同**安全故事 |

---

## 推荐的 PR 组（不要合并不相关的组）

### PR-A — **下一个 PR（推荐）：仅 issue #104**

- **为何是下一个：** 影响范围最小，明确的归属（`tools_extended.go`、`types.go`、tests、README），满足"记录 + 深化 `explain_query`"的需求，不触碰只读保证或连接注册。
- **范围：** 接入 `ExplainQueryInput.Format`（目前未使用），可选的 `EXPLAIN FORMAT=JSON` / `TREE`，规范化结构化输出；扩展 MCP 描述和 README 关于 `run_query` + `EXPLAIN` 的说明。确认所选格式（如 JSON vs TREE）的 **MySQL 版本/语法** 支持，并在不支持时记录**回退**到传统 EXPLAIN。
- **验证：** `go test ./...`，`tools_extended_test.go` 中的扩展模式测试，如果 HTTP 行为变化，还需要 **`http_test.go`** 中的 **`/api/explain`**；按 **workflow.md** 执行 CI 对照检查。

### PR-B — 仅 issue #106（`add_connection`）

- 运行时 DSN 注册、重复名称错误、掩码、可选的 `POST /api/connections`、可选的持久化标志。
- **不要与 #103 捆绑** — 不同的审查关注点（连接信任边界 vs 变更信任边界）。

### PR-C — 仅 issue #103（`write_query`）

- 新的受控工具，仅 DML 验证、确认语义、审计条目、默认禁用。
- **重量级安全/集成测试**；在干净的 `main` 上可在 #106 之前或之后落地，但**不要与 #106 在同一 PR 中**。

**PR-B + PR-C 合并注意事项：** 两者都会触碰共享的注册文件（例如 `main.go`、`types.go`、`tool_wrappers.go`）。落地一个，rebase 另一个——预期会有常规的冲突解决，这不是合并 PR 的理由。

---

## 并行代理规则（实现阶段用）

按照 **AGENTS.md** / **workflow.md** / **dispatching-parallel-agents**：

- **并行：** 例如 README 编辑 vs 仅测试修复 vs 不相关的包 — 在 PR 范围冻结后。
- **顺序：** 共享一个安全设计的任何内容（#103 确认 + 验证器 + 审计）应保持**一个代理或一个协调系列**，不在并行代理之间分割。

---

## 建议的分支名称（示例）

- `feature/issue-104-explain-structured` → **PR-A**
- `feature/issue-106-add-connection` → **PR-B**
- `feature/issue-103-write-query` → **PR-C**

---

## 执行交接

**默认"下一个 PR"：** 从 `main` 的新分支开启 **PR-A (#104)**，运行 **workflow.md** 中的 **CI 对照检查**，然后创建 GitHub PR。仅在 PR 完全满足 issue 时使用 **`Closes #104`**；否则使用 **`Refs #104`**（或拆分为第二个 PR 完成剩余工作）。

如果你想先做 **PR-B** 或 **PR-C**，仅将此文档视为**排序指导**，并相应地更新 PR 描述中的 issue 链接。