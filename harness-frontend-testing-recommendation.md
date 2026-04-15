# 前端验收方案选择：深度对比与推荐

> 整理时间：2026-04-15  
> 背景：方案 A（AI 调 Playwright MCP 实时验证）已有效果但慢；方案 B（埋码协议 + 脚本）领导感兴趣但维护成本高。

---

## 先说结论

**你们的方案 A 方向对，慢的问题有具体解法。**  
**方案 B 的真实维护成本比你们预期的高。**  
**存在第三条路，融合两者优点，是目前业界最主流的生产级做法。**

---

## 一、方案 A 为什么慢：根因分析

来源：[Bug0 — Playwright MCP Build vs Buy](https://bug0.com/blog/playwright-mcp-changes-ai-testing-2026)、[Currents — State of Playwright AI Ecosystem 2026](https://currents.dev/posts/state-of-playwright-ai-ecosystem-in-2026)

**关键数据：**

| 模式 | 每次交互数据量 | 速度对比 |
|------|------------|---------|
| Vision 模式（截图） | 500KB–2MB / 次 | 基准 |
| Snapshot 模式（Accessibility Tree） | 2–5KB / 次 | **10–100x 更快** |

**每个测试任务的 token 消耗：**
- MCP 完整模式：~114,000 tokens / 次
- MCP 增量 snapshot 模式：显著更低
- Playwright CLI + Skills：~27,000 tokens（节省 75%）

**如果你们现在用的是截图分析（vision 模式），单纯切换到 snapshot 模式就能解决大部分速度问题。**

具体配置：
```bash
# 启用增量 snapshot 模式（只传输变化的节点）
PLAYWRIGHT_MCP_SNAPSHOT_MODE=incremental
```

**但速度慢还有第二个更深的根因：**

> "On Day 2... you have to manually debug and re-prompt the agent for every single failure."（Octomind 的研究）

每次验收都重新让 AI 探索、分析、执行，是 O(n) 的成本。**业界的解法是让 AI 只做一次探索，生成稳定的可复用脚本，后续执行脚本而不是重跑 AI 分析。** 这是方案 A 和方案 B 之间的第三条路。

---

## 二、方案 B 的真实成本

来源：[Bug0 — Build vs Buy](https://bug0.com/blog/playwright-mcp-changes-ai-testing-2026)、[Octomind — vs Coding Agents](https://octomind.dev/product/octomind-vs-coding-agents)

**Bug0 的成本测算（基于多个团队案例）：**

| 阶段 | 成本 |
|------|------|
| 初始 prototype（2–4 周） | $8K–$15K |
| 生产就绪（自愈、抖动处理、CI 集成） | $100K–$200K |
| 第一年总计 | $208K–$415K |
| 隐性成本 | 高级工程师变成"测试专员"，**生产效率下降 40%** |

> "Demo works in a sprint; production-ready takes 12 months."

**方案 B 的维护陷阱（Octomind 的总结）：**

> "Coding agents only see code (text). They guess based on syntax."  
> 而运行时能看到的是：**DOM 快照 + 浏览器执行轨迹 + 网络日志**。

手写脚本 + 埋码协议本质上是让开发者用静态文本（代码）来描述动态运行时行为。每次 UI 变化，两套东西都要同步维护：
1. 埋码协议（开发侧）
2. Playwright 脚本（测试侧）

这不是"脚本效率高"，而是把效率从运行时转移到了维护时。

---

## 三、第三条路：AI 生成稳定脚本，脚本负责执行

来源：[TestDino — Playwright AI Ecosystem 2026](https://testdino.com/blog/playwright-ai-ecosystem/)、[Octomind](https://octomind.dev/product/octomind-vs-coding-agents)、[Playwright 官方 Agents](https://playwright.dev/docs/test-agents)

**核心思路：**

```
传统方案 A（你们现在）：
  每次验收 → AI 分析 → AI 执行 → AI 判断（慢，每次都要跑全流程）

传统方案 B：
  开发埋码 → 手写脚本 → CI 执行（快，但维护成本高）

第三条路：
  AI 生成脚本（一次性）→ 脚本 CI 执行（快）→ 脚本失败时 AI 自愈（按需）
```

这是目前业界最主流的生产级方式，有三种实现路径：

---

### 路径 1：Playwright 官方 Agents（Planner → Generator → Healer）

你们已经有方案 A 的基础，迁移成本最低。

**工作流：**
1. **Planner**：AI 探索应用，生成 Markdown 测试计划（人工审查）
2. **Generator**：把计划转成 `.spec.ts`（标准 Playwright 代码，可直接 CI 运行）
3. **CI**：跑生成的脚本，快（不走 AI）
4. **Healer**：脚本挂了才触发 AI 修复（按需，不是每次都跑）

```bash
npx playwright init-agents --loop=opencode  # 你们用 opencode
```

**关键优势**：生成的是标准 Playwright 代码，不绑定任何 AI 服务，CI 执行不消耗 token。

**局限**：Healer 修选择器成功率 75%+，业务逻辑变更需人工介入。

---

### 路径 2：Octomind（托管平台，适合快速落地）

来源：[Octomind 官网](https://octomind.dev/product/octomind-vs-coding-agents)

**核心差异**：不需要你主动触发 Planner，它监听你的部署自动发现新流程：

> "When you merge, Octomind scans your deployment, discovers new flows, and opens a PR with the generated tests automatically."

**生产级能力：**
- AI 看到的是运行时信息（DOM 快照 + 执行轨迹 + 网络日志），不是代码文本
- 自动检测 UI 变化并修复测试
- 托管并行执行环境，无需自建 CI 基础设施
- **生成标准 Playwright 代码，可导出，无厂商锁定**

**适合你们团队的场景**：如果不想自己维护 AI 测试基础设施，这是最轻量的选择。

---

### 路径 3：AgentQL（最小改动，增强方案 B 的稳定性）

来源：[AgentQL 官网](https://www.agentql.com)、[GitHub tinyfish-io/agentql](https://github.com/tinyfish-io/agentql)

如果领导坚持要脚本化方案，这是对方案 B 的最佳增强：

**不需要埋码，用自然语言替代 CSS/XPath Selector：**

```javascript
// 传统脚本（脆弱，埋码才能稳定）
await page.locator('[data-testid="submit-button"]').click();

// AgentQL（无需埋码，AI 理解语义）
const { submitBtn } = await page.queryElements(`{ submitBtn }`);
await submitBtn.click();
```

**解决的核心问题**：方案 B 最大的维护成本是 selector 失效，AgentQL 的自然语言 selector 自愈，UI 怎么变都能找到元素。

**结果**：保留脚本执行效率，去掉埋码协议维护，去掉 selector 维护。

---

## 四、三种方案的实际对比

| 维度 | 你们的方案 A | 方案 B（埋码+脚本） | 第三条路 |
|------|------------|-----------------|---------|
| 每次验收速度 | 慢（每次跑 AI） | 快（直接跑脚本） | **快**（CI 跑生成脚本） |
| 前端开发成本 | 零 | 高（埋码 + 维护协议） | **零** |
| UI 变更适应 | 好（AI 理解语义） | 差（脚本 + 埋码都要改） | **好**（AI 自愈） |
| 维护成本 | 中（每次手动触发） | 高（双重维护） | **低**（自动发现+自愈） |
| 业务逻辑验证 | 好 | 好（需手写断言） | 中（需人工写 AC） |
| 上手成本 | 已有 | 高 | 低（基于已有 A） |

---

## 五、给你们的具体推荐

**你们的核心需求**：速度更快 + 稳定性更好 + 开发成本不增加

**推荐路径：在方案 A 基础上升级到"第三条路 - 路径 1"**

**第一步（本周，解决速度问题）：**
- 确认当前 Playwright MCP 用的是 snapshot 模式而非 vision 模式
- 加 `PLAYWRIGHT_MCP_SNAPSHOT_MODE=incremental`
- 预期效果：速度提升 10x+

**第二步（本月，解决"每次都跑 AI"问题）：**
```bash
npx playwright init-agents --loop=opencode
```
- 让 AI 把现有的验收点生成成 `.spec.ts` 文件
- 把生成的脚本接入 CI，每次 merge 自动跑
- AI 只在脚本失败时介入（Healer）

**第三步（按需，回应领导的脚本诉求）：**
- 展示给领导：CI 里跑的就是标准 Playwright 脚本，执行效率和方案 B 一样
- 区别是：这些脚本不需要埋码，不需要手写，AI 生成并自动维护
- 可以配合 AgentQL 进一步消除 selector 脆弱性

**一句话给领导的说法：**
> 我们不是在用 AI 实时分析，而是用 AI 生成和维护测试脚本，CI 执行的是确定性脚本，速度和手写脚本一样，但开发成本和维护成本都更低。

---

## 六、什么情况下方案 B 才值得做

来源：[Bug0](https://bug0.com/blog/playwright-mcp-changes-ai-testing-2026)

只有同时满足以下三条时：
1. 有 **2+ 名专职测试工程师** 维护测试基础设施
2. 合规要求禁止使用 SaaS 平台（数据不出内网）
3. 有极端定制化需求（行业特定协议、私有化部署）

否则，第三条路是更合理的选择。
