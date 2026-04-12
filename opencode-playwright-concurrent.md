# OpenCode + Playwright MCP 多并发配置指南

---

## 先理解两种 MCP 类型

OpenCode 的 MCP 配置有两种 `type`，对应不同的启动方式：

| type | 行为 | 是否自动启动 | 并发能力 |
|---|---|---|---|
| `local` | OpenCode 启动时自动 fork 子进程（stdio） | 是 | 每个命名实例独立，天然隔离 |
| `remote` | 连接已运行的 SSE HTTP 服务器 | 否（需预先启动） | 单服务器多连接，需 `--isolated` |

**目标：自动启动 + 多并发 → 用 `local` 多实例方案。**

---

## 方案：多命名 local 实例（推荐）

原理：在 `opencode.json` 里注册多个名字不同的 Playwright MCP，每个都是独立的 stdio 进程，OpenCode 启动时全部自动拉起。并发的子 Agent 各自使用不同的实例，互不干扰。

```
OpenCode 启动
  ├── 自动 fork: playwright-a 进程 ──► 独立浏览器 A
  ├── 自动 fork: playwright-b 进程 ──► 独立浏览器 B
  └── 自动 fork: playwright-c 进程 ──► 独立浏览器 C

并发子 Agent:
  ├── subagent-1 使用 playwright-a_* 工具
  ├── subagent-2 使用 playwright-b_* 工具
  └── subagent-3 使用 playwright-c_* 工具
```

---

## 第一步：配置 opencode.json

在项目根目录创建（或修改）`opencode.json`：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "playwright-a": {
      "type": "local",
      "command": ["npx", "-y", "@playwright/mcp@latest", "--isolated", "--headless"],
      "enabled": true,
      "timeout": 30000
    },
    "playwright-b": {
      "type": "local",
      "command": ["npx", "-y", "@playwright/mcp@latest", "--isolated", "--headless"],
      "enabled": true,
      "timeout": 30000
    },
    "playwright-c": {
      "type": "local",
      "command": ["npx", "-y", "@playwright/mcp@latest", "--isolated", "--headless"],
      "enabled": true,
      "timeout": 30000
    }
  }
}
```

**关键参数说明：**
- `type: "local"` → OpenCode 自动启动，无需手动操作
- `--isolated` → 浏览器状态保存在内存，不写磁盘，进程退出即清理
- `--headless` → 无头模式，适合服务器/后台运行
- `timeout: 30000` → 浏览器操作超时 30 秒（默认 5 秒对复杂页面太短）
- `-y` → npx 自动安装，不提示确认

启动 OpenCode 后，工具名称会变成：
- `playwright-a_browser_navigate`、`playwright-a_browser_click` ...
- `playwright-b_browser_navigate`、`playwright-b_browser_click` ...
- `playwright-c_browser_navigate`、`playwright-c_browser_click` ...

---

## 第二步：配置子 Agent

在 `.opencode/agents/` 目录下为每个并发任务创建子 Agent 配置文件。

### .opencode/agents/browser-agent-a.json

```json
{
  "name": "browser-agent-a",
  "description": "浏览器自动化子 Agent，使用 playwright-a 实例",
  "mode": "subagent",
  "model": "claude-sonnet-4-6",
  "steps": 50,
  "permission": {
    "mcp_playwright-b": "deny",
    "mcp_playwright-c": "deny"
  },
  "prompt": ".opencode/prompts/browser-agent.md"
}
```

### .opencode/agents/browser-agent-b.json

```json
{
  "name": "browser-agent-b",
  "description": "浏览器自动化子 Agent，使用 playwright-b 实例",
  "mode": "subagent",
  "model": "claude-sonnet-4-6",
  "steps": 50,
  "permission": {
    "mcp_playwright-a": "deny",
    "mcp_playwright-c": "deny"
  },
  "prompt": ".opencode/prompts/browser-agent.md"
}
```

### .opencode/agents/browser-agent-c.json

```json
{
  "name": "browser-agent-c",
  "description": "浏览器自动化子 Agent，使用 playwright-c 实例",
  "mode": "subagent",
  "model": "claude-sonnet-4-6",
  "steps": 50,
  "permission": {
    "mcp_playwright-a": "deny",
    "mcp_playwright-b": "deny"
  },
  "prompt": ".opencode/prompts/browser-agent.md"
}
```

**`permission` 的作用**：把其他两个 playwright 实例的工具设为 `deny`，强制每个子 Agent 只能用自己的浏览器实例，避免混用。

### 共用 Prompt 文件：.opencode/prompts/browser-agent.md

```markdown
你是一个浏览器自动化 Agent。

## 工具使用规则
- 只使用你被分配的 playwright 实例（其他实例的工具对你不可用）
- 每次导航前先检查当前页面状态
- 操作失败重试不超过 3 次，超过后报告错误并停止

## 输出规范
任务完成后，输出结构化结果：
- 访问的 URL
- 提取的数据
- 遇到的错误（如有）
```

---

## 第三步：主 Agent 编排并发调用

在 OpenCode 中，主 Agent 通过 `@agent名称` 派发并发任务：

```
同时调用 @browser-agent-a、@browser-agent-b、@browser-agent-c，
分别抓取以下页面的产品价格：
- agent-a: https://site-a.com/products
- agent-b: https://site-b.com/products  
- agent-c: https://site-c.com/products

三个任务并发执行，完成后汇总价格对比表。
```

OpenCode 会并发派发三个子 Agent，每个使用独立的浏览器实例。

---

## 目录结构总览

```
项目根目录/
├── opencode.json                        # MCP 注册（playwright-a/b/c）
└── .opencode/
    ├── agents/
    │   ├── browser-agent-a.json         # 子 Agent A，绑定 playwright-a
    │   ├── browser-agent-b.json         # 子 Agent B，绑定 playwright-b
    │   └── browser-agent-c.json         # 子 Agent C，绑定 playwright-c
    └── prompts/
        └── browser-agent.md             # 共用行为规范
```

---

## 全局配置（所有项目通用）

如果希望对所有项目生效，将 `opencode.json` 中的 MCP 配置放到全局：

```
~/.config/opencode/config.json           # 全局 opencode 配置
~/.config/opencode/agents/               # 全局 Agent 配置
```

全局 `config.json` 内容与项目级 `opencode.json` 格式相同。

---

## 常见问题

### Q：想用有界面的浏览器（调试时）

去掉 `--headless` 参数：

```json
"command": ["npx", "-y", "@playwright/mcp@latest", "--isolated"]
```

### Q：需要更多并发数

直接在 `opencode.json` 里加更多命名实例（playwright-d、playwright-e...），对应增加子 Agent 配置文件。每个实例约占 100-200MB 内存。

### Q：想用 Chrome 而不是默认的 Chromium

```json
"command": ["npx", "-y", "@playwright/mcp@latest", "--isolated", "--headless", "--browser", "chrome"]
```

需要本机已安装 Chrome。

### Q：如何验证 MCP 已启动

OpenCode 启动后，输入 `/mcp` 查看已加载的 MCP 工具列表，应该能看到 `playwright-a_*`、`playwright-b_*`、`playwright-c_*` 三组工具。
