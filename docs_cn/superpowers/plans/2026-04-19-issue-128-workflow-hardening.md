# CI/Release 工作流加固实施计划

> **针对代理工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 来按任务实施此计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 修复 Issue #128 中识别的七个 CI/Release 工作流缺陷 — 预发布 QA 门禁、格式强制执行、集成测试阻塞、健康轮询、lint 版本锁定、Action 版本对齐，以及过期文档。

**架构：** 所有更改都在 `.github/workflows/` YAML 文件和 `workflow.md` 中。没有 Go 代码更改。每个任务都可以通过 `gh workflow view` 或直接检查来独立测试。

**技术栈：** GitHub Actions、GoReleaser、golangci-lint、gofmt、bash

---

## 文件映射

| 文件 | 更改 |
|------|---------|
| `.github/workflows/release.yml` | 添加 `pre-release-check` 作业；将 `goreleaser` 设置为依赖于此；将 `build-push-action` 对齐到 `@v6` |
| `.github/workflows/qa.yml` | 修复格式检查；扩展 `qa-summary` 失败策略；替换 `sleep 3`；锁定 lint 版本；将 `build-push-action` 对齐到 `@v6` |
| `workflow.md` | 从待办中删除过期的 issue #104 |

---

### 任务 1：修复 qa.yml 中的空操作格式检查
- 将 `go fmt ./... || true` 替换为实际的 `gofmt -l` 强制执行。
- 提交信息：`fix(ci): 用 gofmt -l 格式检查强制执行替代空操作的 go fmt`

### 任务 2：将 golangci-lint 锁定到特定版本
- 将 `golangci-lint-action@v4` 更新为 `@v6`，并将 `version: latest` 更改为 `version: v1.64.8`。
- 提交信息：`fix(ci): 将 golangci-lint 锁定到 v1.64.8 以防止意外破坏`

### 任务 3：在 api-tests 中用健康轮询替换 sleep 3
- 将固定的 `sleep 3` 替换为一个重试循环，轮询 `localhost:9306/health`，最多重试 20 次。
- 提交信息：`fix(ci): 在 api-tests 中用健康轮询循环替换 sleep 3`

### 任务 4：扩展 qa-summary 失败策略以包含集成测试
- 在失败条件中添加 `integration-tests` 和 `integration-tests-mariadb`。
- 提交信息：`fix(ci): 将集成测试包含在 qa-summary 失败条件中`

### 任务 5：在 qa.yml 中将 docker/build-push-action 对齐到 @v6
- 将 `docker/build-push-action@v5` 更新为 `@v6`。
- 提交信息：`fix(ci): 在 qa.yml 中将 docker/build-push-action 对齐到 @v6`

### 任务 6：向 release.yml 添加 pre-release-check 作业
- 添加一个新作业，运行单元测试和集成测试，然后再启动 GoReleaser。
- 将 `goreleaser` 作业设置为 `needs: pre-release-check`。
- 提交信息：`fix(ci): 添加 pre-release-check 作业；将 goreleaser 设置为依赖于单元测试和集成测试`

### 任务 7：从 workflow.md 待办中删除过期的 issue #104
- 删除 issue #104 行，更新"最近已交付"行以包含 #104。
- 提交信息：`docs: 从 workflow.md 待办中删除过期的 issue #104（已在 PR #126 中合并）`

### 任务 8：创建 PR
- 分支：`fix/issue-128-workflow-hardening`
- PR 标题：`fix(ci): 工作流加固 — 预发布门禁、格式强制执行、摘要策略、lint 锁定`
- 关联：`Closes #128`