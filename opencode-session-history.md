# OpenCode 历史会话查找与导出

## 问题背景

使用 `/session` 命令只能看到近期会话，更早的历史记录无法通过 UI 访问。

## 根本原因

不是数据被删除，而是前端硬编码了展示窗口限制：

```typescript
// packages/app/src/context/global-sync/types.ts
SESSION_RECENT_WINDOW = 4 * 60 * 60 * 1000  // 只展示 4 小时内的会话
SESSION_RECENT_LIMIT  = 50                    // 最多展示 50 条
```

服务端没有时间限制，历史数据完整保留在磁盘上。

## 数据存储位置

| 安装方式 | 数据路径 |
|----------|----------|
| CLI（npm 安装） | `%LOCALAPPDATA%\opencode\storage\` |
| 桌面版（Electron） | `%APPDATA%\ai.opencode.desktop\opencode\storage\` |

数据格式：JSON 文件（旧版）或 SQLite 数据库 `opencode.db`（新版），目录结构：

```
storage/
  session/       # 会话元数据
  message/       # 消息内容
  part/          # 消息分段
  session_diff/  # 文件变更记录
```

## 查找数据目录的方法

npm 安装的 OpenCode，先确认 Node 环境变量：

```bash
node -e "const os=require('os'); console.log(process.env.LOCALAPPDATA); console.log(os.homedir())"
```

然后在对应路径下查找：

```bash
find "$LOCALAPPDATA" -maxdepth 5 -type d -name "opencode"
find "$APPDATA" -maxdepth 5 -type d -name "opencode"
```

## 让 UI 显示更多历史的方案

**方案一：改源码重新构建**（成本高）

修改 `packages/app/src/context/global-sync/types.ts` 里的 `SESSION_RECENT_WINDOW` 常量，改为更大的时间窗口后重新编译。

**方案二：直接读取原始数据**（推荐）

数据完整保留在磁盘，直接读取 JSON 或查询 SQLite，导出为 Markdown 作为写作素材，比折腾 UI 更实用。

找到 `storage/` 目录后：

- JSON 格式：直接读 `message/` 下的文件
- SQLite 格式：用以下命令查询

```bash
sqlite3 opencode.db ".tables"
sqlite3 opencode.db "SELECT * FROM session ORDER BY created_at DESC LIMIT 20;"
```
