# Playwright MCP 调试上下文原则

## 工具能力

以下两个工具始终可用，无需额外配置，**必须主动调用**才返回数据：

- `browser_console_messages` — 获取页面控制台输出（error/warning/info/debug）
- `browser_network_requests` — 获取页面所有网络请求，支持 URL 正则过滤

## 使用原则

**1. 触发操作后立即取数据**
每次点击按钮或提交表单后，紧接着调用上面两个工具。不要等到最后才查，页面跳转后记录会重置。

**2. 网络请求用 filter 精确过滤**
只看接口请求，过滤掉静态资源：
```
filter: "/api/.*"
```
需要看请求体时传 `requestBody: true`，需要看请求头传 `requestHeaders: true`。

**3. 控制台只看 error**
验收时传 `level: "error"`，排查时传 `level: "info"`。

**4. 登录 session 不自动恢复**
`--isolated` 模式下每次连接都是全新 Context，需要先执行登录步骤。登录成功后才执行验收操作。

**5. 截图留存**
关键步骤（操作前/操作后/报错时）调用截图工具，方便留档。

## 标准验收步骤模板

```
1. 打开 [URL]，如果跳转登录页，先完成登录
2. 执行 [具体操作：填写表单/点击按钮等]
3. 调用 browser_network_requests，filter="/api/.*"，requestBody=true
   → 报告：接口URL、状态码、响应体（重点看 code/message 字段）
4. 调用 browser_console_messages，level="error"
   → 报告：所有 error 输出
5. 截图当前页面状态
6. 结论：操作是否符合预期，接口是否正常
```
