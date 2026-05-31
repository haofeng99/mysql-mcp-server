# 发布检查清单

发布通过 GitHub Actions 和 GoReleaser 自动化完成。本文档是维护者的完整检查清单。

## 工作流程（人类和编码代理）

不要在 `main` 分支上直接提交面向用户的行为变更后就撒手不管。在同一个工作流中或之前：

1. **Issue** — 在 GitHub 上创建一个描述变更的 issue（或一个带有检查清单的发布跟踪 issue）。
2. **PR** — 优先使用引用 issue 的 Pull Request。如果代码已在没有 PR 的情况下合并到 `main`，请创建一个**回顾性 issue** 并链接提交记录和即将添加的 CHANGELOG 部分（保证可追溯性优于分支操作）。
3. **CHANGELOG** — 在打标签**之前**添加带版本的章节（`[1.7.0-rc.N]` 或 GA）；按照[预发布](#预发布)部分，保持 `[Unreleased]` 位于顶部。
4. **标签** — 带注释的标签 + `git push origin vX.Y.Z`，以便 GoReleaser 运行。

跳过 CHANGELOG 直到 RC 标签推送后再补会向用户和 Homebrew 传递错误信息；跳过 issue 会让发布说明难以解释。

## 候选发布版本

- 标签格式：**`v1.7.0-rc.1`**、**`v1.7.0-rc.2`** 等（语义化版本预发布，位于补丁段之后）。
- 在 **`CHANGELOG.md`** 中添加 **`[1.7.0-rc.N] - 日期`** 章节（参见现有的 **[Unreleased]** / RC 标题）。
- **`goreleaser`** 的 `release.prerelease: auto` 在标签指示 RC 时将 GitHub Release 标记为 **预发布**。
- 验证后，以 **GA** 方式发布 **`v1.7.0`**（或按下面的章节将 RC 说明合并到 `v1.7.0 - 日期` 中）。

```bash
git tag -a v1.7.0-rc.1 -m "候选发布版本 v1.7.0-rc.1"
git push origin v1.7.0-rc.1
```

## 预发布

1. **更新 CHANGELOG**
   - 将 `[Unreleased]` 内容移动到带版本的章节下，例如 `## vX.Y.Z - YYYY-MM-DD`。
   - 如果从 RC 转为 GA：将 RC 标题改为 GA（例如 `[1.6.0.rc1] - 日期` → `v1.6.0 - 日期`）。
   - 在顶部添加一个新的空白 `## [Unreleased]` 章节。
2. **提交** CHANGELOG（及其他发布相关更改）。

## 执行发布

3. **打标签并推送**（使用你在 CHANGELOG 中记录的版本）：

   ```bash
   git tag vX.Y.Z
   git push origin vX.Y.Z
   ```

   推送标签将触发 **Release** 工作流。它将：
   - 运行 GoReleaser：构建二进制文件，创建带有 changelog 的 GitHub Release，更新 Homebrew formula（如果设置了 `HOMEBREW_TAP_TOKEN`）。
   - 构建并推送 Docker 镜像到 `ghcr.io/askdba/mysql-mcp-server`。
   - 更新 `main` 分支上的 README 版本徽章（update-readme 作业）。

## 发布后

4. **`main` 上的 CHANGELOG**  
   如果发布是从非 main 分支进行的，默认分支（main）可能仍显示 RC 或旧标题。请更新它以反映已发布的版本：
   - 检出 `main`，拉取最新版本。
   - 在 `CHANGELOG.md` 中，将 RC 或预发布标题替换为 GA 行（例如 `## v1.6.0 - YYYY-MM-DD`）。
   - 提交并推送到 `main`。

5. **Homebrew tap**
   - **Formula：** 当 `HOMEBREW_TAP_TOKEN` 设置后，GoReleaser 会自动更新。确保该密钥已配置（在 [askdba/homebrew-tap](https://github.com/askdba/homebrew-tap) 上具有 `contents: write` 权限的 token）。
   - **Tap README：** 不会自动更新。发布后，如有需要请更新 [askdba/homebrew-tap](https://github.com/askdba/homebrew-tap) 的 README（版本引用、功能、安装/升级说明），然后提交并推送到 tap 的 `main` 分支。

6. **本地安装**（为你自己或用户）：  
   `brew update && brew upgrade mysql-mcp-server`

## 手动更新 formula（如需要）

如果 GoReleaser 没有推送 formula 或你需要修复它：

```bash
cd /path/to/homebrew-tap
# 编辑 Formula/mysql-mcp-server.rb（版本、URL、SHA）
git add Formula/mysql-mcp-server.rb
git commit -m "mysql-mcp-server: 更新到 vX.Y.Z"
git push origin main
```

## 参考

- Release 工作流：[.github/workflows/release.yml](../.github/workflows/release.yml)
- GoReleaser 配置：[.goreleaser.yml](../.goreleaser.yml)