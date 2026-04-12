# 自动化验收与 Bug 修复验证：工具选型分析

## 你的具体场景

- 功能点 UI 验收
- Bug 修复验证
- 需要登录（保持会话）
- 查看前端控制台输出
- 点击按钮等交互操作
- 查看接口报错详情（Network 面板）

---

## 三个工具的本质差异

| | Playwright MCP | Browser Use | Agent Browser（Steel 等）|
|---|---|---|---|
| **本质** | MCP 工具服务，Agent 精确调用 | Python Agent 框架，LLM 自主决策 | 浏览器基础设施层 |
| **控制方式** | 确定性：指令 → 精确执行 | 非确定性：任务 → LLM 自行规划步骤 | API 调用，需自行集成 |
| **集成方式** | 直接作为 MCP 工具用 | 独立 Python 程序 | REST API / SDK |
| **登录持久化** | 原生支持（storage-state） | 支持，但配置复杂 | 支持（session 管理） |
| **控制台访问** | 原生支持（--caps devtools） | 不是主要特性 | 需额外配置 |
| **Network 面板** | 原生支持（--caps devtools） | 不支持 | 部分支持 |
| **适合场景** | 确定性验证、已知步骤的测试 | 探索式任务、不确定路径 | 大规模并发基础设施 |

---

## 结论：你的场景用 Playwright MCP

原因：

1. **查看接口报错** → 需要 Network 面板 → 只有 Playwright MCP 原生支持（`--caps devtools`）
2. **查看前端控制台** → 只有 Playwright MCP 有 `--console-level` 参数
3. **登录保持** → Playwright MCP 的 `--storage-state` 专门解决这个问题
4. **验收和验证** → 你知道要验什么，是确定性操作，不需要 LLM 自主决策
5. **已在 OpenCode 里** → 不需要额外部署 Python 环境

Browser Use 适合的是"帮我去网上查某个信息"这类探索式任务，不适合"验证这个按钮点击后接口是否返回 200"这类确定性验证。

---

## Playwright MCP 针对你场景的完整配置

### opencode.json

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "playwright": {
      "type": "local",
      "command": [
        "npx", "-y", "@playwright/mcp@latest",
        "--isolated",
        "--caps", "devtools",
        "--console-level", "info",
        "--storage-state", ".playwright/session.json",
        "--output-dir", ".playwright/output",
        "--timeout-action", "10000",
        "--timeout-navigation", "30000"
      ],
      "enabled": true,
      "timeout": 60000
    }
  }
}
```

**参数说明：**

| 参数 | 作用 | 你的场景 |
|---|---|---|
| `--caps devtools` | 开启 DevTools 工具集（控制台 + Network） | 查接口报错、看 console |
| `--console-level info` | 捕获 info 及以上级别的控制台输出 | 看前端日志，改成 `error` 只看报错 |
| `--storage-state .playwright/session.json` | 登录状态持久化文件 | 登录一次，后续复用 |
| `--output-dir .playwright/output` | 截图、日志保存位置 | 验收存档 |
| `--timeout-action 10000` | 单次操作超时 10 秒 | 防止卡住 |
| `--timeout-navigation 30000` | 页面导航超时 30 秒 | 应对慢页面 |

### .gitignore 追加

```
.playwright/session.json    # 登录 session，不提交
.playwright/output/         # 截图输出，不提交
```

---

## 使用流程

### 第一次：执行登录并保存 session

让 Agent 做一次登录，session 自动写入 `session.json`：

```
用 playwright 工具：
1. 打开 http://your-app.com/login
2. 输入用户名 xxx，密码 xxx，点击登录
3. 等待跳转到首页，确认登录成功
（session 已自动保存，后续启动无需重新登录）
```

之后每次重启 OpenCode，Playwright MCP 读取 `session.json`，直接进入已登录状态。

---

### 日常：UI 功能验收

```
用 playwright 工具验收"新增订单"功能：
1. 打开 http://your-app.com/orders/new
2. 截图当前状态
3. 填写表单：商品名称"测试商品"，数量 2，收货地址"北京市..."
4. 点击"提交订单"按钮
5. 捕获点击后的所有网络请求，找到 POST /api/orders 请求
6. 报告该接口的：请求参数、响应状态码、响应体
7. 截图最终页面状态
8. 检查页面是否出现"订单创建成功"提示
```

---

### 日常：Bug 修复验证

```
验证 Bug #456 修复情况（用户点击删除按钮后接口 500）：

用 playwright 工具：
1. 打开 http://your-app.com/items
2. 找到列表中任意一项，点击"删除"按钮
3. 捕获删除操作触发的网络请求（DELETE /api/items/*）
4. 报告：
   - 请求 URL 和方法
   - 响应状态码（期望 200，之前是 500）
   - 响应体完整内容
   - 页面是否有错误提示
5. 同时报告点击过程中控制台是否有 error 级别输出
```

---

### 接口报错排查

DevTools 能力开启后，可以访问完整的 Network 信息：

```
用 playwright 工具：
1. 打开 http://your-app.com/dashboard
2. 执行触发报错的操作（点击"导出报表"按钮）
3. 获取所有网络请求列表，找出状态码非 2xx 的请求
4. 对每个失败请求，报告：
   - 完整 URL
   - 请求方法和请求体
   - 响应状态码
   - 响应体（重点看 error message 字段）
   - 请求耗时
5. 同时列出控制台所有 error 和 warning 输出
```

---

## session.json 过期了怎么办

登录 session 通常几小时到几天后失效。失效后 Agent 调用会看到重定向到登录页。

**处理方式**：在 AGENTS.md 或 prompt 里写明：

```markdown
## Playwright 登录规则

如果打开任何页面后被重定向到 /login，说明 session 已过期，执行以下步骤：
1. 在登录页输入账号 [账号] 密码 [密码]
2. 登录成功后继续原任务
（新 session 会自动覆盖写入 .playwright/session.json）
```

---

## 并发验收多个功能点

结合之前的多实例配置，每个子 Agent 验收一个功能模块：

```json
"mcp": {
  "playwright-a": {
    "type": "local",
    "command": ["npx", "-y", "@playwright/mcp@latest",
      "--isolated", "--caps", "devtools", "--console-level", "info",
      "--storage-state", ".playwright/session-a.json",
      "--output-dir", ".playwright/output-a"
    ]
  },
  "playwright-b": {
    "type": "local",
    "command": ["npx", "-y", "@playwright/mcp@latest",
      "--isolated", "--caps", "devtools", "--console-level", "info",
      "--storage-state", ".playwright/session-b.json",
      "--output-dir", ".playwright/output-b"
    ]
  }
}
```

各实例有独立的 session 文件，登录状态互不影响。
