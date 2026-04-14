# OpenCode MySQL MCP 使用指南

本地开发，连接远程 MySQL，读写权限，凭证不进代码。

---

## 第一步：MySQL 建专用账号

在你的远程 MySQL 上执行（用 root 或管理员账号登录后）：

```sql
-- 创建专用账号，只给业务需要的权限
CREATE USER 'ai_dev'@'%' IDENTIFIED BY '你的强密码';
GRANT SELECT, INSERT, UPDATE ON 你的数据库名.* TO 'ai_dev'@'%';
-- 如果需要建表/改表结构，加上：
-- GRANT CREATE, ALTER, DROP ON 你的数据库名.* TO 'ai_dev'@'%';
FLUSH PRIVILEGES;
```

> 不要用 root 账号连，AI 执行 SQL 无需 `DELETE` 全表或 `DROP DATABASE` 的权限。

---

## 第二步：设置 Windows 系统环境变量

**方式 A：当前终端临时生效（重开终端失效）**

```powershell
$env:MYSQL_DSN = "mysql://ai_dev:你的强密码@远程主机IP:3306/你的数据库名"
```

**方式 B：永久写入用户环境变量（推荐）**

```powershell
[System.Environment]::SetEnvironmentVariable(
  "MYSQL_DSN",
  "mysql://ai_dev:你的强密码@远程主机IP:3306/你的数据库名",
  "User"
)
```

执行后重启终端或 opencode 生效。

**DSN 格式说明：**

```
mysql://用户名:密码@主机IP或域名:端口/数据库名

# 示例
mysql://ai_dev:MyP@ssw0rd@192.168.1.100:3306/myapp

# 开启 SSL（远程库建议加）
mysql://ai_dev:MyP@ssw0rd@192.168.1.100:3306/myapp?tls=true
```

---

## 第三步：在 opencode.json 启用 MCP

打开项目根目录的 `opencode.json`，把 `dbhub` 的 `enabled` 改为 `true`：

```json
"dbhub": {
  "type": "local",
  "command": [
    "powershell", "-NoProfile", "-Command",
    "npx -y @bytebase/dbhub@0.12.1 --transport stdio --dsn $env:MYSQL_DSN"
  ],
  "enabled": true,
  "timeout": 15000
}
```

> `opencode.json` 里没有密码，`$env:MYSQL_DSN` 在 PowerShell 运行时才展开，安全。

---

## 第四步：验证连接

启动 opencode，在对话框输入：

```
列出数据库里所有的表
```

如果能返回表名列表，说明连接成功。

**连接失败排查：**

```powershell
# 确认环境变量已设置
echo $env:MYSQL_DSN

# 手动测试 dbhub 能否启动（Ctrl+C 退出）
npx @bytebase/dbhub@0.12.1 --transport stdio --dsn $env:MYSQL_DSN
```

---

## 可用工具和示例提问

dbhub 提供两个工具，opencode 会自动调用，你直接用自然语言描述需求即可。

**浏览结构（`search_objects`）：**
```
列出所有表
描述 orders 表的结构
orders 表有哪些索引？
```

**执行 SQL（`execute_sql`）：**
```
查询最近 7 天的订单数量，按天分组
把 user_id 为 123 的用户状态改为 active
在 users 表里新增一个 last_login_at 字段，类型 datetime
统计每个用户的订单总金额，取前 10 名
```

---

## 注意事项

- `opencode.json` 里不要填真实密码，凭证只存在系统环境变量里
- 如果 `opencode.json` 已被 git 追踪，确保文件里没有密码才提交
- AI 执行写操作前不会二次确认，确保 MySQL 账号权限是你愿意授予的范围
- 操作生产库前，建议先在开发库验证 AI 生成的 SQL
