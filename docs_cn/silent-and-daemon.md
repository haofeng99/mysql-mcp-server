# 静默与守护进程模式

本文档介绍如何以最小控制台输出（静默模式）和后台进程（守护进程模式）运行 mysql-mcp-server，适用于生产环境和服务管理器。

## 静默模式（`-s` / `--silent`）

当设置 `--silent`（或 `-s`）时：

- **INFO** 和 **WARN** 日志消息被抑制。
- **ERROR** 消息仍然输出到 stderr，以确保故障和诊断信息可见。

在以下场景使用静默模式：

- 在 systemd、launchd 或其他单独记录日志的进程管理器下运行时。
- 减少容器或 CI 日志中的噪音。
- 服务器已通过健康检查或审计日志进行监控。

**示例：**

```bash
mysql-mcp-server --silent --config /etc/mysql-mcp-server/config.yaml
MYSQL_MCP_HTTP=1 mysql-mcp-server --silent --config /path/to/config.yaml
```

结构化（JSON）日志不受 `--silent` 对消息**级别**的影响：如果 `MYSQL_MCP_JSON_LOGS=1`，仅跳过 INFO/WARN 行；ERROR 行仍以 JSON 格式输出。

## 守护进程模式（`-d` / `--daemon`）

在 **Unix** 上设置 `--daemon`（或 `-d`）时：

- 进程 **fork** 一个子进程来运行服务器。
- **父进程** 立即退出（退出码 0）。
- **子进程** 在新的会话中运行（与终端分离），并使用相同的配置继续运行。

守护进程模式适用于 **HTTP REST API 模式**（`MYSQL_MCP_HTTP=1`）。在 stdio MCP 中使用也是可行的，但不太常见（子进程将从 stdin 读取并向 stdout 写入，无终端）。

**示例：**

```bash
# 在后台启动 HTTP 服务器
MYSQL_MCP_HTTP=1 mysql-mcp-server --daemon --config /path/to/config.yaml

# 配合静默模式（在作为守护进程运行时推荐）
MYSQL_MCP_HTTP=1 mysql-mcp-server --daemon --silent --config /path/to/config.yaml
```

**Windows：** `--daemon` 是一个空操作。请使用 Windows Service 或在后台运行进程（例如在单独的终端中或通过调度程序）。

## 服务管理器模板

仓库中包含了在进程管理器下运行服务器的示例单元文件：

| 平台   | 文件                                                                 | 描述                    |
|-----------|----------------------------------------------------------------------|--------------------------------|
| systemd   | [contrib/systemd/mysql-mcp-server.service](contrib/systemd/mysql-mcp-server.service) | 示例 systemd 单元           |
| launchd   | [contrib/launchd/com.askdba.mysql-mcp-server.plist](contrib/launchd/com.askdba.mysql-mcp-server.plist) | 示例 launchd plist（macOS）  |

### systemd（Linux）

1. 复制单元文件：  
   `sudo cp contrib/systemd/mysql-mcp-server.service /etc/systemd/system/`
2. 编辑路径和环境变量：  
   `sudo edit /etc/systemd/system/mysql-mcp-server.service`  
   设置 `ExecStart` 为你的二进制文件和配置路径。可选择使用 `EnvironmentFile=` 来设置 `MYSQL_*` 变量。
3. 重新加载并启用：  
   `sudo systemctl daemon-reload`  
   `sudo systemctl enable --now mysql-mcp-server`

### launchd（macOS）

1. 复制 plist：  
   `cp contrib/launchd/com.askdba.mysql-mcp-server.plist ~/Library/LaunchAgents/`  
   （或 `/Library/LaunchDaemons/` 用于系统级服务）
2. 编辑 `ProgramArguments` 和路径（二进制、配置、日志路径）。
3. 加载并启动：  
   `launchctl load ~/Library/LaunchAgents/com.askdba.mysql-mcp-server.plist`

## 总结

| 标志 / 选项   | 效果 |
|-----------------|--------|
| `-s` / `--silent` | 抑制 INFO 和 WARN；ERROR 仍输出到 stderr |
| `-d` / `--daemon` | Fork 并分离（Unix）；父进程退出，子进程运行服务器 |
| contrib/systemd  | Linux 的示例 systemd 单元 |
| contrib/launchd  | macOS 的示例 launchd plist |

要查看完整的 CLI 选项，请运行 `mysql-mcp-server --help`。