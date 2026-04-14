# OpenCode 本地数据库 MCP 配置指南

> 以下 MCP 均支持本地 stdio 运行，数据不出本机。

---

## 方案对比

| MCP | 支持数据库 | 安装 | 特点 |
|-----|-----------|------|------|
| [bytebase/dbhub](https://github.com/bytebase/dbhub) | PG / MySQL / SQL Server / MariaDB / SQLite | `npx @bytebase/dbhub` | 零依赖，token 高效，支持自定义 SQL 模板 |
| [@modelcontextprotocol/server-postgres](https://github.com/modelcontextprotocol/servers/tree/main/src/postgres) | PostgreSQL | `npx @modelcontextprotocol/server-postgres` | Anthropic 官方，只读模式，安全 |
| [@modelcontextprotocol/server-sqlite](https://github.com/modelcontextprotocol/servers/tree/main/src/sqlite) | SQLite | `npx @modelcontextprotocol/server-sqlite` | Anthropic 官方，支持写入，含 memo 功能 |
| [FreePeak/db-mcp-server](https://github.com/FreePeak/db-mcp-server) | PG / MySQL / SQLite / Oracle / TimescaleDB | Go 二进制 | 多库并发连接，按库生成独立工具 |
| [executeautomation/mcp-database-server](https://github.com/executeautomation/mcp-database-server) | SQLite / SQL Server / PG / MySQL | `npx mcp-database-server` | .NET 实现，工具最全 |
| [designcomputer/mysql_mcp_server](https://github.com/designcomputer/mysql_mcp_server) | MySQL | `uvx mysql-mcp-server` | Python，MySQL 专用 |
| [RichardHan/mssql_mcp_server](https://github.com/RichardHan/mssql_mcp_server) | SQL Server | `uvx mssql-mcp-server` | SQL Server 专用，支持列表/读/写 |

---

## 推荐：dbhub（最通用）

**工具能力：**
- `execute_sql` — 执行任意 SQL（含 DDL/DML，可配置只读模式）
- `search_objects` — 浏览表、列、索引、存储过程
- 自定义 SQL 模板（`dbhub.toml`，支持参数化复用）

**opencode.json 配置：**

```json
"dbhub": {
  "type": "local",
  "command": [
    "npx", "-y", "@bytebase/dbhub@latest",
    "--transport", "stdio",
    "--dsn", "postgres://user:password@localhost:5432/mydb"
  ],
  "enabled": true,
  "timeout": 15000
}
```

各数据库 DSN 格式：
```
# PostgreSQL
postgres://user:password@localhost:5432/dbname

# MySQL
mysql://user:password@localhost:3306/dbname

# SQL Server
sqlserver://user:password@localhost:1433?database=dbname

# SQLite（本地文件路径）
sqlite:///path/to/database.db
```

**只读模式（推荐生产环境）：**
```json
"command": [
  "npx", "-y", "@bytebase/dbhub@latest",
  "--transport", "stdio",
  "--dsn", "postgres://user:password@localhost:5432/mydb",
  "--readonly"
]
```

---

## Anthropic 官方：PostgreSQL MCP

只读，安全，适合只需要查询的场景。

```json
"postgres": {
  "type": "local",
  "command": [
    "npx", "-y", "@modelcontextprotocol/server-postgres",
    "postgresql://user:password@localhost:5432/mydb"
  ],
  "enabled": true,
  "timeout": 10000
}
```

---

## Anthropic 官方：SQLite MCP

支持读写，内置 memo（持久化记忆）功能。

```json
"sqlite": {
  "type": "local",
  "command": [
    "npx", "-y", "@modelcontextprotocol/server-sqlite",
    "--db-path", "/path/to/database.db"
  ],
  "enabled": true,
  "timeout": 10000
}
```

---

## 多数据库并发：FreePeak/db-mcp-server

适合同时操作多个库（如 MySQL 业务库 + PG 分析库）。

```json
"db-mcp": {
  "type": "local",
  "command": ["./db-mcp-server"],
  "environment": {
    "DB_CONFIGS": "[{\"id\":\"pg1\",\"type\":\"postgres\",\"dsn\":\"postgres://...\"},{\"id\":\"mysql1\",\"type\":\"mysql\",\"dsn\":\"mysql://...\"}]"
  },
  "enabled": true,
  "timeout": 15000
}
```

每个库自动生成独立工具：`query_pg1`、`execute_mysql1`、`schema_pg1` 等。

---

## 安全建议

1. **不要把密码明文写进 opencode.json**，用环境变量替代：
   ```json
   "environment": { "DB_PASSWORD": "从shell环境继承" }
   ```
   或在命令里引用 `$DB_DSN`（需要 shell 展开支持）

2. 生产/只读库走 `--readonly` 模式，防止 AI 误写数据

3. SQLite 本地开发最简单，不需要启动任何服务
