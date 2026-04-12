# Playwright MCP 多 Agent 并发调用指南

---

## 核心问题：为什么默认配置不能并发

Playwright MCP 默认使用 **stdio 模式**：

```
Agent 进程 ──stdio──► Playwright MCP 进程 ──► 一个浏览器实例
```

stdio 是一对一的管道，天然不支持多个 Agent 同时调用。  
多 Agent 并发需要每个 Agent 拥有**独立的浏览器上下文**。

---

## 三种方案对比

| 方案 | 隔离级别 | 资源消耗 | 配置复杂度 | 适用场景 |
|---|---|---|---|---|
| A. SSE 单服务器 + isolated | 浏览器 Context 隔离 | 低（共享一个浏览器进程） | 简单 | 推荐，通用 |
| B. 多 SSE 实例 | 浏览器进程隔离 | 高 | 中等 | 需要完全隔离 |
| C. stdio + Agent 工具 | 进程级隔离 | 中 | 最简单 | Claude Code Agent 工具场景 |

---

## 方案 A：SSE 单服务器（推荐）

### 原理

```
Agent-1 ──HTTP──┐
Agent-2 ──HTTP──┼──► Playwright MCP Server（port 3000）──► Browser
Agent-3 ──HTTP──┘         每个连接独立 Context（--isolated）
```

`--isolated` 的含义：每个 HTTP 客户端连接进来，都会分配一个独立的浏览器 Context，互不干扰，也不写磁盘。

### 第一步：启动 Playwright MCP 服务器

```bash
# 基础启动（无头模式，适合服务器）
npx @playwright/mcp@latest --port 3000 --isolated --headless

# 有头模式（本地调试）
npx @playwright/mcp@latest --port 3000 --isolated

# 指定浏览器（默认 chromium）
npx @playwright/mcp@latest --port 3000 --isolated --browser chrome

# 完整推荐参数
npx @playwright/mcp@latest \
  --port 3000 \
  --isolated \
  --headless \
  --host 127.0.0.1 \
  --snapshot-mode full \
  --output-dir ./playwright-output
```

### 第二步：配置 Claude Code settings.json

```json
// ~/.claude/settings.json
{
  "mcpServers": {
    "playwright": {
      "type": "sse",
      "url": "http://localhost:3000/sse"
    }
  }
}
```

**注意**：`type: "sse"` 是关键，告诉 Claude Code 用 HTTP 连接而不是 stdio 启动子进程。

### 第三步：验证多连接隔离

启动服务器后，用 curl 验证它在跑：

```bash
curl http://localhost:3000/sse
# 应该看到 SSE 流开始，而不是报错
```

### 持久化运行（Windows 服务）

开发环境用 PM2 保持后台运行：

```bash
npm install -g pm2

# 启动并命名
pm2 start "npx @playwright/mcp@latest --port 3000 --isolated --headless" \
  --name playwright-mcp

# 开机自启
pm2 save
pm2 startup

# 查看状态
pm2 status
pm2 logs playwright-mcp
```

---

## 方案 B：多 SSE 实例（完全隔离）

当你需要多个 Agent **彻底独立**（独立的浏览器进程、独立的 Cookie/Session），且想精确控制"哪个 Agent 用哪个浏览器"时使用。

### 启动多个实例

```bash
# 实例 1（用于需求分析 Agent）
npx @playwright/mcp@latest --port 3001 --isolated --headless &

# 实例 2（用于方案验证 Agent）
npx @playwright/mcp@latest --port 3002 --isolated --headless &

# 实例 3（用于测试执行 Agent）
npx @playwright/mcp@latest --port 3003 --isolated --headless &
```

或者用 PM2 批量管理：

```yaml
# ecosystem.config.js
module.exports = {
  apps: [
    {
      name: "playwright-mcp-1",
      script: "npx",
      args: "@playwright/mcp@latest --port 3001 --isolated --headless",
    },
    {
      name: "playwright-mcp-2",
      script: "npx",
      args: "@playwright/mcp@latest --port 3002 --isolated --headless",
    },
    {
      name: "playwright-mcp-3",
      script: "npx",
      args: "@playwright/mcp@latest --port 3003 --isolated --headless",
    }
  ]
}
```

```bash
pm2 start ecosystem.config.js
```

### 配置 settings.json

```json
// ~/.claude/settings.json
{
  "mcpServers": {
    "playwright-1": {
      "type": "sse",
      "url": "http://localhost:3001/sse"
    },
    "playwright-2": {
      "type": "sse",
      "url": "http://localhost:3002/sse"
    },
    "playwright-3": {
      "type": "sse",
      "url": "http://localhost:3003/sse"
    }
  }
}
```

**使用方式**：在不同 Agent 的 prompt 中明确指定使用哪个 MCP：

```
// Agent 1 的指令
你只能使用 playwright-1 工具执行浏览器操作...

// Agent 2 的指令  
你只能使用 playwright-2 工具执行浏览器操作...
```

---

## 方案 C：stdio 模式（最简，适合 Agent 工具）

当你使用 Claude Code 的 `Agent` 工具派生子 Agent 时，**每个子 Agent 是独立的进程**，stdio 模式天然就能做到进程级隔离。

```
主 Agent
  ├── 派生 Agent-1 ──stdio──► Playwright 进程 1 ──► 浏览器 1
  ├── 派生 Agent-2 ──stdio──► Playwright 进程 2 ──► 浏览器 2
  └── 派生 Agent-3 ──stdio──► Playwright 进程 3 ──► 浏览器 3
```

每个 Agent 进程在启动时，会 fork 一个新的 Playwright MCP 进程。

### 配置 settings.json

```json
// ~/.claude/settings.json 或项目 .claude/settings.json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest", "--isolated"],
      "type": "stdio"
    }
  }
}
```

`type: "stdio"` 可以省略，这是默认值。

### 适用场景示例

```
主 Agent prompt：
"并发派生 3 个子 Agent，分别用 playwright 工具：
 - Agent A：访问竞品网站 A，抓取价格数据
 - Agent B：访问竞品网站 B，抓取价格数据
 - Agent C：访问竞品网站 C，抓取价格数据
最后汇总三份数据"
```

派生的三个 Agent 各自拥有独立的 Playwright 进程，完全并发，互不影响。

### 缺点

每个 Agent 启动时都要 fork 一次 `npx`（冷启动约 3-10 秒），并发数多时浏览器进程数会线性增长，内存消耗大。

---

## 各方案 settings.json 完整示例

```json
// 方案 A：SSE 单服务器（先手动启动服务器）
{
  "mcpServers": {
    "playwright": {
      "type": "sse",
      "url": "http://localhost:3000/sse"
    }
  }
}
```

```json
// 方案 B：多 SSE 实例（先手动启动多个服务器）
{
  "mcpServers": {
    "playwright-1": {
      "type": "sse",
      "url": "http://localhost:3001/sse"
    },
    "playwright-2": {
      "type": "sse",
      "url": "http://localhost:3002/sse"
    },
    "playwright-3": {
      "type": "sse",
      "url": "http://localhost:3003/sse"
    }
  }
}
```

```json
// 方案 C：stdio 自动 fork（无需手动启动服务器）
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": [
        "@playwright/mcp@latest",
        "--isolated",
        "--headless"
      ]
    }
  }
}
```

---

## 常见问题

### Q1：多个 Agent 同时操作同一个 URL 会冲突吗？

不会。`--isolated` 保证每个连接有独立的 Context，Cookie、Session、页面状态完全隔离，即使访问同一个 URL 也互不影响。

### Q2：SSE 模式下，一个 Agent 关掉后，浏览器 Context 会自动销毁吗？

会。HTTP 连接断开时，对应的浏览器 Context 会被 Playwright MCP 自动清理。

### Q3：`--shared-browser-context` 是什么意思，什么时候用？

与 `--isolated` 相反，所有连接**共享同一个 Context**（共享 Cookie、登录状态、localStorage）。

适用场景：多个 Agent 需要共享同一个已登录的浏览器状态（比如一个 Agent 登录，另一个 Agent 用已登录的状态继续操作）。

**并发场景不要用这个**，多个 Agent 同时操作同一个 Context 会互相干扰。

### Q4：并发时 port 3000 的服务器撑得住多少个 Agent？

Playwright MCP 的瓶颈在浏览器 Context 数量，不在网络连接。每个 Context 约占 50-150MB 内存。一台 8GB 内存机器，稳定并发约 10-20 个 Agent。

超过这个量，改用方案 B（多实例）+ 多台机器水平扩展。

### Q5：settings.json 放项目目录还是用户目录？

```
~/.claude/settings.json          → 对所有项目生效（全局）
./项目根目录/.claude/settings.json → 只对当前项目生效（推荐）
```

多代码仓场景，建议在父目录或专用的 harness 目录放一份全局配置，各仓库不用各自配置。

---

## 推荐配置（总结）

绝大多数多 Agent 并发场景，用这个组合：

```bash
# 1. 启动（一次）
npx @playwright/mcp@latest --port 3000 --isolated --headless

# 2. settings.json
```

```json
{
  "mcpServers": {
    "playwright": {
      "type": "sse",
      "url": "http://localhost:3000/sse"
    }
  }
}
```

这样配置后，每次 Claude Code 派生新的 Agent 工具调用，都会建立一个新的 HTTP 连接，自动拿到独立的浏览器 Context，无需任何额外处理。
