# OpenCode Playwright MCP 隔离配置

## opencode.json 配置

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "playwright": {
      "type": "local",
      "command": [
        "npx", "-y", "@playwright/mcp@latest",
        "--isolated"
      ],
      "enabled": true,
      "timeout": 30000
    }
  }
}
```

`--isolated` 是实现隔离的唯一关键参数，其余不需要。

## 隔离机制（源码级说明）

Playwright MCP `program.js` 的核心逻辑：

```js
// --isolated 触发共享浏览器进程
const useSharedBrowser = config.sharedBrowserContext || config.browser.isolated;

// 每个连接调用 newContext()，创建独立 Context
const browserContext = config.browser.isolated
  ? await browser.newContext(config.browser.contextOptions)  // 新 Context
  : browser.contexts()[0];                                   // 共享 Context
```

效果：
- **一个浏览器进程**（一个窗口）
- **每个 MCP 连接独立一个 Context**（独立的 Cookie、Storage、Session）
- **不写磁盘**（进程退出即清理，不留残留状态）

## 与不加 `--isolated` 的区别

| | 加 `--isolated` | 不加 |
|---|---|---|
| 浏览器进程 | 一个（共享） | 每个连接各一个 |
| 浏览器窗口 | 一个 | 多个 |
| Cookie/Session | 每个连接独立 | 复用上次状态 |
| 磁盘残留 | 无 | 有（user-data-dir） |

## `type: "local"` 的自动启动行为

OpenCode 启动时自动 fork Playwright MCP 进程，无需手动操作。每次重启 OpenCode，旧的浏览器 Context 销毁，新的全新 Context 创建。

如需有界面的浏览器（可见窗口）：不加 `--headless` 即可，上面配置默认就是有界面的。

如需无头模式（服务器环境）：加 `--headless` 参数。

## 验证隔离是否生效

OpenCode 中输入 `/mcp`，看到 `playwright_browser_*` 工具列表即说明已加载。

两次独立任务之间，登录状态不会延续——这是隔离生效的直接表现。
