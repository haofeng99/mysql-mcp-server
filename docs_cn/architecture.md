# MySQL MCP Server 架构

本文档提供 MySQL MCP Server 的详细架构图。

## 目录

- [高层架构](#高层架构)
- [组件结构](#组件结构)
- [请求流程](#请求流程)
  - [MCP 模式](#mcp-模式-stdio)
  - [HTTP REST API 模式](#http-rest-api-模式)
  - [Metrics HTTP 边车（stdio）](#metrics-http-边车-stdio)
- [配置加载](#配置加载)
- [连接管理](#连接管理)
- [工具分类](#工具分类)

---

## 高层架构

MySQL MCP Server 充当 AI 客户端与 MySQL 数据库之间的桥梁，支持通过 **stdio** 的 MCP 协议、可选的 **HTTP REST API**（设置 `MYSQL_MCP_HTTP=1`），以及可选的 **仅 metrics HTTP 监听器**（设置 `MYSQL_MCP_METRICS_HTTP=1`，在 stdio MCP 基础上提供 `/health`、`/status`、`/api/metrics/tokens`，监听 `MYSQL_HTTP_PORT`）。

```mermaid
graph TB
    subgraph "AI 客户端"
        Claude[("Claude Desktop<br/>(MCP 客户端)")]
        Cursor[("Cursor IDE<br/>(MCP 客户端)")]
        ChatGPT[("ChatGPT<br/>自定义 GPT")]
        HTTPClient[("HTTP 客户端<br/>curl, Postman")]
        Browser[("浏览器 / curl<br/>(/status 仪表盘)")]
    end
    
    subgraph "MySQL MCP Server"
        direction TB
        MCP["MCP 协议处理器<br/>(stdio JSON-RPC)"]
        HTTP["HTTP REST API<br/>(完整 API; MYSQL_MCP_HTTP)"]
        MetricsHTTP["Metrics HTTP<br/>(可选; MYSQL_MCP_METRICS_HTTP)"]
        Tools["工具处理器<br/>(核心、扩展、向量)"]
        CM["连接管理器<br/>(每个 DSN 一个连接池)"]
        Config["配置<br/>(文件 + 环境变量)"]
    end
    
    subgraph "MySQL 数据库"
        DB1[("生产环境<br/>MySQL 8.x/9.x")]
        DB2[("预发布环境<br/>MySQL 8.x")]
        DB3[("开发环境<br/>MySQL/MariaDB")]
    end
    
    Claude -->|"stdio<br/>MCP 协议"| MCP
    Cursor -->|"stdio<br/>MCP 协议"| MCP
    ChatGPT -->|"HTTP/HTTPS"| HTTP
    HTTPClient -->|"HTTP/HTTPS"| HTTP
    Browser -->|"GET /status, /health"| MetricsHTTP
    
    MCP --> Tools
    HTTP --> Tools
    MetricsHTTP -.->|"与 MCP<br/>同进程"| MCP
    Tools --> CM
    Config -.->|"加载"| CM
    
    CM -->|"mysql 驱动<br/>+TLS"| DB1
    CM -->|"mysql 驱动"| DB2
    CM -->|"mysql 驱动"| DB3
```

---

## 组件结构

代码库按清晰的职责划分为多个包。

```mermaid
graph TB
    subgraph "cmd/mysql-mcp-server"
        main["main.go<br/>入口点，MCP 服务端设置"]
        tools["tools.go<br/>核心工具处理器"]
        toolsExt["tools_extended.go<br/>扩展工具处理器"]
        http["http.go<br/>REST API 处理器"]
        conn["connection.go<br/>连接管理器"]
        types["types.go<br/>输入/输出类型"]
        logging["logging.go<br/>结构化日志"]
        tokenEst["token_estimator.go<br/>令牌计数"]
    end
    
    subgraph "internal/config"
        config["config.go<br/>配置加载"]
        file["file.go<br/>文件配置解析器"]
    end
    
    subgraph "internal/mysql"
        client["client.go<br/>MySQL 客户端封装"]
    end
    
    subgraph "internal/api"
        middleware["middleware.go<br/>HTTP 中间件"]
        ratelimit["ratelimit.go<br/>速率限制器"]
        response["response.go<br/>响应辅助工具"]
    end
    
    subgraph "internal/util"
        validator["sql_validator.go<br/>SQL 验证"]
        parser["sql_parser.go<br/>SQL 解析"]
        identifiers["identifiers.go<br/>标识符引用"]
    end
    
    main --> tools
    main --> toolsExt
    main --> http
    main --> conn
    tools --> types
    toolsExt --> types
    http --> types
    
    main --> config
    config --> file
    
    conn --> client
    tools --> client
    toolsExt --> client
    
    http --> middleware
    http --> ratelimit
    http --> response
    
    tools --> validator
    tools --> parser
    tools --> identifiers
```

---

## 请求流程

### MCP 模式（stdio）

在 MCP 模式下，服务器通过 stdin/stdout 使用 JSON-RPC 通信。

```mermaid
sequenceDiagram
    participant Client as AI 客户端<br/>(Claude/Cursor)
    participant MCP as MCP 服务器<br/>(main.go)
    participant Tool as 工具处理器<br/>(tools.go)
    participant CM as 连接<br/>管理器
    participant MySQL as MySQL<br/>数据库

    Client->>MCP: JSON-RPC 请求<br/>{"method": "tools/call", "params": {...}}
    MCP->>MCP: 解析请求<br/>识别工具
    
    alt 工具: mysql_query
        MCP->>Tool: 执行 mysql_query
        Tool->>Tool: 验证 SQL<br/>(只读检查)
        Tool->>CM: 获取连接
        CM->>MySQL: 执行查询
        MySQL-->>CM: 结果行
        CM-->>Tool: 返回结果
        Tool->>Tool: 格式化响应<br/>(应用行限制)
        Tool-->>MCP: 工具结果
    else 工具: list_tables
        MCP->>Tool: 执行 list_tables
        Tool->>CM: 获取连接
        CM->>MySQL: SHOW TABLES
        MySQL-->>CM: 表列表
        CM-->>Tool: 返回表
        Tool-->>MCP: 工具结果
    end
    
    MCP-->>Client: JSON-RPC 响应<br/>{"result": {...}}
```

### HTTP REST API 模式

在 HTTP 模式下，服务器暴露 RESTful 端点。

```mermaid
sequenceDiagram
    participant Client as HTTP 客户端<br/>(ChatGPT/curl)
    participant MW as 中间件<br/>(速率限制、日志)
    participant Handler as HTTP 处理器<br/>(http.go)
    participant Tool as 工具处理器<br/>(tools.go)
    participant CM as 连接<br/>管理器
    participant MySQL as MySQL<br/>数据库

    Client->>MW: POST /api/query<br/>{"sql": "SELECT...", "database": "mydb"}
    MW->>MW: 检查速率限制
    
    alt 触发速率限制
        MW-->>Client: 429 Too Many Requests<br/>Retry-After: 1
    else 允许
        MW->>Handler: 转发请求
        Handler->>Handler: 解析 JSON 正文
        Handler->>Tool: 执行 mysql_query
        Tool->>Tool: 验证 SQL
        Tool->>CM: 获取连接
        CM->>MySQL: 执行查询
        MySQL-->>CM: 结果行
        CM-->>Tool: 返回结果
        Tool-->>Handler: 工具结果
        Handler-->>MW: JSON 响应
        MW-->>Client: 200 OK<br/>{"success": true, "data": {...}}
    end
```

### Metrics HTTP 边车（stdio）

当 `MYSQL_MCP_METRICS_HTTP=1` 且 `MYSQL_MCP_HTTP` 不是主要的全 REST 模式时，进程保持 **stdio MCP** 并在 `MYSQL_HTTP_PORT`（默认 **9306**）上启动一个小型 HTTP 服务器，用于 `/health`、`/api/metrics/tokens` 和 `/status`。这与同一 MCP 工具调用（例如 Claude Desktop）共享 **token 指标**。全 REST 模式（`MYSQL_MCP_HTTP=1`）会取代此边车。

```mermaid
sequenceDiagram
    participant Claude as AI 客户端<br/>(stdio MCP)
    participant MCP as MCP 服务器<br/>(main.go)
    participant Browser as 浏览器 / curl
    participant MH as Metrics HTTP<br/>(http.go)

    Claude->>MCP: tools/call (JSON-RPC)
    MCP-->>Claude: 工具结果（已记录 token）

    Browser->>MH: GET /status
    MH-->>Browser: HTML 仪表盘（相同 token 计数器）

    Note over MCP,MH: 同一操作系统进程；metrics 监听器为可选
```

---

## 配置加载

配置从多个来源加载，具有明确的优先级顺序。

```mermaid
flowchart TB
    Start([开始]) --> FindConfig{配置文件<br/>是否存在？}
    
    FindConfig -->|"是"| LoadFile["加载配置文件<br/>(YAML/JSON)"]
    FindConfig -->|"否"| UseDefaults["使用默认值"]
    
    LoadFile --> ApplyEnv["应用环境变量<br/>覆盖"]
    UseDefaults --> ApplyEnv
    
    ApplyEnv --> LoadConns{连接<br/>来源？}
    
    LoadConns -->|"MYSQL_CONNECTIONS"| ParseJSON["解析 JSON 数组"]
    LoadConns -->|"MYSQL_DSN*"| ParseDSN["解析 DSN 环境变量"]
    LoadConns -->|"配置文件"| UseFileConns["使用文件连接"]
    
    ParseJSON --> ApplySSL["将 MYSQL_SSL<br/>应用到没有<br/>显式 SSL 的连接"]
    ParseDSN --> ApplySSL
    UseFileConns --> Validate
    
    ApplySSL --> Validate{配置是否<br/>有效？}
    
    Validate -->|"是"| Ready([配置就绪])
    Validate -->|"否"| Error([错误：无 DSN])
    
    subgraph "优先级（从高到低）"
        direction LR
        P1["1. 环境变量"]
        P2["2. 配置文件"]
        P3["3. 默认值"]
        P1 --> P2 --> P3
    end
```

### 配置文件搜索顺序

服务器按以下顺序搜索配置文件（先找到的优先）：

```mermaid
flowchart TB
    subgraph "1. 显式指定（最高优先级）"
        A["--config 标志"]
        B["MYSQL_MCP_CONFIG 环境变量"]
    end
    
    subgraph "2. 当前目录"
        C["./mysql-mcp-server.yaml"]
        D["./mysql-mcp-server.yml"]
        E["./mysql-mcp-server.json"]
    end
    
    subgraph "3. 用户配置"
        F["~/.config/mysql-mcp-server/config.yaml"]
        G["~/.config/mysql-mcp-server/config.yml"]
        H["~/.config/mysql-mcp-server/config.json"]
    end
    
    subgraph "4. 系统配置（最低优先级）"
        I["/etc/mysql-mcp-server/config.yaml"]
        J["/etc/mysql-mcp-server/config.yml"]
        K["/etc/mysql-mcp-server/config.json"]
    end
    
    A --> C
    B --> C
    C --> D --> E --> F --> G --> H --> I --> J --> K
    K --> L(["先找到的优先"])
```

---

## 连接管理

连接管理器处理多个数据库连接，支持连接池。

```mermaid
graph TB
    subgraph "连接管理器"
        CM["ConnectionManager<br/>- connections map<br/>- activeConn string<br/>- mutex"]
    end
    
    subgraph "连接池"
        Pool1["池: default<br/>MaxOpen: 10<br/>MaxIdle: 5"]
        Pool2["池: production<br/>MaxOpen: 10<br/>MaxIdle: 5"]
        Pool3["池: staging<br/>MaxOpen: 10<br/>MaxIdle: 5"]
    end
    
    subgraph "连接池配置"
        direction LR
        MaxOpen["MaxOpenConns<br/>(MYSQL_MAX_OPEN_CONNS)"]
        MaxIdle["MaxIdleConns<br/>(MYSQL_MAX_IDLE_CONNS)"]
        Lifetime["ConnMaxLifetime<br/>(MYSQL_CONN_MAX_LIFETIME_MINUTES)"]
        IdleTime["ConnMaxIdleTime<br/>(MYSQL_CONN_MAX_IDLE_TIME_MINUTES)"]
    end
    
    subgraph "MySQL 服务器"
        DB1[("localhost:3306<br/>default")]
        DB2[("prod-server:3306<br/>production")]
        DB3[("staging:3306<br/>staging")]
    end
    
    CM --> Pool1
    CM --> Pool2
    CM --> Pool3
    
    Pool1 -->|"SSL: skip-verify"| DB1
    Pool2 -->|"SSL: true"| DB2
    Pool3 -->|"SSL: false"| DB3
    
    MaxOpen -.-> Pool1
    MaxIdle -.-> Pool1
    Lifetime -.-> Pool1
    IdleTime -.-> Pool1
```

### 连接选择流程

```mermaid
flowchart TB
    Request["工具请求"] --> HasConn{是否指定<br/>连接？}
    
    HasConn -->|"是"| FindConn["查找命名连接"]
    HasConn -->|"否"| UseActive["使用活动连接"]
    
    FindConn --> Exists{连接是否<br/>存在？}
    Exists -->|"是"| GetPool["获取连接池"]
    Exists -->|"否"| Error["错误：未知连接"]
    
    UseActive --> GetPool
    GetPool --> Execute["执行查询"]
    Execute --> Return["返回结果"]
```

---

## 工具分类

工具按功能分为三类。

```mermaid
graph TB
    subgraph "核心工具（始终可用）"
        direction LR
        mysql_query["mysql_query<br/>执行只读 SQL"]
        list_databases["list_databases<br/>显示所有数据库"]
        list_tables["list_tables<br/>显示数据库中的表"]
        describe_table["describe_table<br/>显示表结构"]
        ping["ping<br/>测试连接"]
        server_info["server_info<br/>MySQL 版本信息"]
        list_connections["list_connections<br/>显示所有 DSN"]
        use_connection["use_connection<br/>切换活动 DSN"]
    end
    
    subgraph "扩展工具（MYSQL_MCP_EXTENDED=1）"
        direction LR
        list_indexes["list_indexes"]
        show_create_table["show_create_table"]
        explain_query["explain_query"]
        list_views["list_views"]
        list_triggers["list_triggers"]
        list_procedures["list_procedures"]
        list_functions["list_functions"]
        list_partitions["list_partitions"]
        get_database_size["get_database_size"]
        get_table_sizes["get_table_sizes"]
        list_foreign_keys["list_foreign_keys"]
        show_status["show_status"]
        show_variables["show_variables"]
    end
    
    subgraph "向量工具（MYSQL_MCP_VECTOR=1）"
        direction LR
        vector_search["vector_search<br/>相似度搜索"]
        get_vector_info["get_vector_info<br/>向量列信息"]
    end
    
    Core["启用：<br/>默认"] --> mysql_query
    Extended["启用：<br/>MYSQL_MCP_EXTENDED=1"] --> list_indexes
    Vector["启用：<br/>MYSQL_MCP_VECTOR=1"] --> vector_search
```

---

## 安全架构

```mermaid
graph TB
    subgraph "输入验证"
        SQLValid["SQL 验证器<br/>- 只读强制<br/>- 危险语句拦截"]
        IdentValid["标识符验证器<br/>- 正确引用<br/>- 注入防护"]
    end
    
    subgraph "连接安全"
        TLS["TLS/SSL 选项<br/>- true: 验证证书<br/>- skip-verify: 不验证<br/>- preferred: 映射到 skip-verify"]
        DSNMask["DSN 掩码<br/>- 日志中隐藏密码"]
    end
    
    subgraph "API 安全"
        RateLimit["速率限制<br/>- 按 IP 跟踪<br/>- 可配置 RPS/burst"]
        AuditLog["审计日志<br/>- 查询日志<br/>- 连接跟踪"]
    end
    
    Request["入站请求"] --> SQLValid
    SQLValid --> IdentValid
    IdentValid --> TLS
    TLS --> RateLimit
    RateLimit --> AuditLog
    AuditLog --> Execute["执行查询"]
```

---

## 部署选项

```mermaid
graph TB
    subgraph "部署方式"
        Binary["二进制<br/>Homebrew / 直接下载"]
        Docker["Docker<br/>ghcr.io/askdba/mysql-mcp-server"]
        Source["源码编译<br/>go install"]
    end
    
    subgraph "集成模式"
        MCPMode["MCP 模式 (stdio)<br/>Claude Desktop, Cursor"]
        HTTPMode["HTTP 模式<br/>ChatGPT, REST 客户端"]
    end
    
    subgraph "配置来源"
        EnvVars["环境变量"]
        ConfigFile["配置文件 (YAML/JSON)"]
        CLI["命令行参数"]
    end
    
    Binary --> MCPMode
    Binary --> HTTPMode
    Docker --> MCPMode
    Docker --> HTTPMode
    Source --> MCPMode
    Source --> HTTPMode
    
    EnvVars --> Binary
    EnvVars --> Docker
    ConfigFile --> Binary
    ConfigFile --> Docker
```

关于以后台服务方式运行服务器（如 systemd 或 launchd）并减少日志输出，请参阅[静默与守护进程模式](silent-and-daemon.md)。

---

## 数据流总结

```mermaid
flowchart LR
    subgraph 输入
        AI["AI 客户端"]
        HTTP["HTTP 客户端"]
    end
    
    subgraph 处理
        Parse["解析请求"]
        Validate["验证"]
        Route["路由到工具"]
        Execute["执行"]
        Format["格式化响应"]
    end
    
    subgraph 输出
        Response["响应"]
    end
    
    AI --> Parse
    HTTP --> Parse
    Parse --> Validate
    Validate --> Route
    Route --> Execute
    Execute --> Format
    Format --> Response
```

---

## 参见

- [静默与守护进程模式](silent-and-daemon.md) — `--silent`、`--daemon` 及服务管理器模板（systemd、launchd）。