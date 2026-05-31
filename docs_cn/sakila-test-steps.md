<!-- 与 ../test-steps.md（仓库根目录）保持同步。编辑任一文件并同步更新。 -->

1. 测试步骤（Sakila 集成测试矩阵）

前置条件
- Docker 引擎正在运行
- Bash（辅助函数使用 POSIX 标准的 `while`、`sleep` 和算术运算）
- 仓库根目录：`<repo-root>` — 替换为你的克隆路径，或运行 `cd` 到包含 `docker-compose.test.yml` 的目录（在 `cd` 之后等同于 `$(pwd)`）

### Bash 辅助函数（每个 shell 会话执行一次；需要先 `cd` 到 `<repo-root>`）

```bash
wait_docker_healthy() {
  local cname=$1
  local max_attempts=${2:-60}
  local sleep_s=${3:-2}
  local i=0
  while [ "$i" -lt "$max_attempts" ]; do
    st=$(docker inspect -f '{{.State.Health.Status}}' "$cname" 2>/dev/null || echo "unknown")
    echo "  $cname 健康状态: $st (尝试 $((i+1))/$max_attempts)"
    [ "$st" = "healthy" ] && return 0
    i=$((i+1))
    sleep "$sleep_s"
  done
  echo "错误: $cname 在 $((max_attempts * sleep_s))s 内未变为健康状态" >&2
  return 1
}

wait_mysqladmin_ping() {
  local name=$1
  local max_attempts=${2:-60}
  local sleep_s=${3:-3}
  local i=0
  while [ "$i" -lt "$max_attempts" ]; do
    echo "  $name: mysqladmin ping (尝试 $((i+1))/$max_attempts)"
    if docker exec "$name" mysqladmin ping -h localhost -u root -ptestpass 2>/dev/null; then
      echo "  $name: MySQL 已就绪。"
      return 0
    fi
    i=$((i+1))
    sleep "$sleep_s"
  done
  echo "错误: $name 在 $((max_attempts * sleep_s))s 内未响应 mysqladmin ping" >&2
  return 1
}
```

步骤（在 `<repo-root>` 目录下执行）

1) 从 compose 启动 MySQL 8.4
   docker compose -f docker-compose.test.yml up -d mysql84

2) 等待 mysql84 变为健康状态（`mysql-mcp-test-84`）
   wait_docker_healthy mysql-mcp-test-84

3) 在 MySQL 8.4 上运行 Sakila 测试
   MYSQL_SAKILA_DSN="mcpuser:mcppass00@tcp(localhost:3307)/sakila?parseTime=true" \
     go test -tags=integration ./tests/integration -run Sakila -v

4) 从 compose 启动 MySQL 9.0
   docker compose -f docker-compose.test.yml up -d mysql90

5) 确保 mysql90 为健康状态（`mysql-mcp-test-90`）
   wait_docker_healthy mysql-mcp-test-90

6) 在 MySQL 9.0 上运行 Sakila 测试
   MYSQL_SAKILA_DSN="mcpuser:mcppass00@tcp(localhost:3308)/sakila?parseTime=true" \
     go test -tags=integration ./tests/integration -run Sakila -v

7) 从 compose 启动 MariaDB 11.4
   docker compose -f docker-compose.test.yml up -d mariadb11

8) 确保 mariadb11 为健康状态（`mysql-mcp-test-mariadb-11`）
   wait_docker_healthy mysql-mcp-test-mariadb-11

9) 在 MariaDB 11.4 上运行 Sakila 测试
   MYSQL_SAKILA_DSN="mcpuser:mcppass00@tcp(localhost:3310)/sakila?parseTime=true" \
     go test -tags=integration ./tests/integration -run Sakila -v

10) 在备用端口上运行 MySQL 8.0（3306 已被占用）
    docker run -d --name mysql-mcp-test-80-alt \
      -e MYSQL_ROOT_PASSWORD=testpass \
      -e MYSQL_DATABASE=testdb \
      -e MYSQL_USER=testuser \
      -e MYSQL_PASSWORD=testpass \
      -p 3311:3306 \
      -v mysql80_alt_data:/var/lib/mysql \
      -v "$(pwd)/tests/sql/init.sql":/docker-entrypoint-initdb.d/01-init.sql:ro \
      -v "$(pwd)/tests/sql/sakila-schema.sql":/docker-entrypoint-initdb.d/02-sakila-schema.sql:ro \
      -v "$(pwd)/tests/sql/sakila-data.sql":/docker-entrypoint-initdb.d/03-sakila-data.sql:ro \
      mysql:8.0 \
      --default-authentication-plugin=mysql_native_password \
      --character-set-server=utf8mb4 \
      --collation-server=utf8mb4_unicode_ci

11) 等待 mysql-mcp-test-80-alt 就绪（重试直到 mysqladmin ping 成功）
    wait_mysqladmin_ping mysql-mcp-test-80-alt

12) 在 MySQL 8.0（备用端口）上运行 Sakila 测试
    MYSQL_SAKILA_DSN="mcpuser:mcppass00@tcp(localhost:3311)/sakila?parseTime=true" \
      go test -tags=integration ./tests/integration -run Sakila -v

清理（选项 2）

13) 停止并删除 compose 容器、网络和卷
    docker compose -f docker-compose.test.yml down -v

14) 删除 MySQL 8.0 备用容器和卷
    docker rm -f mysql-mcp-test-80-alt
    docker volume rm mysql80_alt_data