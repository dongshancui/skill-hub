# 前端 Harness 自动化验收：业界主流方式全景

> 整理时间：2026-04-14  
> 来源：Playwright 官方文档、Anthropic Engineering、VS Code 团队博客、TestDino、Cegeka、Ministry of Testing 等

---

## 先说结论：你们的两种方式都有迹可循，但各有明确的边界

**方式 A（AI 自主调用 Playwright MCP 验证）**：这是目前最主流的方向，VS Code 团队、Anthropic 都在用，但你们遇到的"效率和稳定性不够"是普遍痛点，根因有解。

**方式 B（开发时埋码 + Playwright 脚本共用协议）**：这个思路与业界"Accessibility Tree 优先"的方向高度吻合，但业界的做法不是自定义埋码协议，而是直接用 ARIA 语义属性作为协议标准。

---

## 一、业界的四种主流方式

### 方式 1：Playwright 官方三 Agent 体系（2025.10 起正式发布）

来源：[Playwright 官方文档 — Test Agents](https://playwright.dev/docs/test-agents)、[TestDino — Playwright AI Ecosystem 2026](https://testdino.com/blog/playwright-ai-ecosystem/)

Playwright v1.56（2025-10）正式内置三个 AI Agent：

```
Planner   ← 探索应用，产出 Markdown 测试计划（人可读，可审查）
    ↓
Generator ← 把 Markdown 计划转成可执行的 .spec.ts 文件
    ↓
Healer    ← 运行测试套件，自动修复失败（选择器失效、超时等）
```

**初始化命令（支持三种 agent loop）：**
```bash
npx playwright init-agents --loop=claude      # Claude Code
npx playwright init-agents --loop=vscode      # VS Code Copilot
npx playwright init-agents --loop=opencode    # OpenCode
```

生成的目录结构：
```
specs/          ← 人工可读的 Markdown 测试计划
tests/          ← 自动生成的 .spec.ts，与 spec 一一对应
.github/agents/ ← Agent 定义文件
.vscode/mcp.json
```

**Healer 的修复能力**：选择器失效（75%+ 成功率）、等待条件、超时调整。**不擅长**：业务逻辑变化、API contract 变更、多租户逻辑。

**直接 LLM-MCP vs 官方 Agents 的对比：**

| 维度 | 直接 LLM + Playwright MCP | 官方 Playwright Agents |
|------|--------------------------|----------------------|
| 控制权 | 完全自定义 | 标准化流程，灵活性低 |
| 稳定性 | 依赖 LLM 不幻觉 | 内置规划+验证+修复 |
| 延迟 | 低（直接通信） | 高（多步骤开销） |
| 维护成本 | 自己维护 prompt | 官方维护 |

---

### 方式 2：Accessibility Tree 优先 + Semantic Locator（取代埋码）

来源：[Playwright Tips — Why You Should Avoid data-testid](https://www.weekly.playwright-user-event.org/tip-13-why-you-should-avoid-data-testid-and-use-aria-label-instead)、[TestDino — Playwright AI Ecosystem](https://testdino.com/blog/playwright-ai-ecosystem/)

**这是对你们"埋码方式"的业界答案**：不需要自定义埋码协议，直接用 ARIA 语义属性作为标准协议。

> "The entire Playwright AI ecosystem is built on this tree, not on screenshots."

**Playwright MCP 默认的 snapshot 模式读取 Accessibility Tree：**
```yaml
# MCP 返回的 YAML 结构示例
role: button
name: "提交订单"
state: enabled
```

这意味着：**只要你的前端组件有正确的语义属性，AI 就能理解和操作它**——不需要另外维护一套埋码协议。

**data-testid vs aria-label 的选择：**

| 属性 | 适用场景 | AI 可读性 | 用户可读性 |
|------|---------|----------|----------|
| `aria-label` | 首选，有语义意义 | ✅ 直接映射到 Accessibility Tree | ✅ 屏幕阅读器可用 |
| `role` + `name` | 标准组件 | ✅ | ✅ |
| `data-testid` | 最后手段，无语义场景 | ⚠️ 仅作定位 | ❌ |

**业界判断标准：**
> "Would an AI or a screen reader know what this element does?" —— 如果答案是否，就需要加语义属性。

**Playwright 推荐的 Locator 优先级（与 AI agent 高度兼容）：**
1. `getByRole()` ← 最稳定
2. `getByLabel()`
3. `getByText()`
4. `getByTestId()` ← 最后选项

---

### 方式 3：AI Agent Loop + Playwright MCP（VS Code 和 Anthropic 的做法）

来源：[VS Code — How VS Code Builds with AI](https://code.visualstudio.com/blogs/2026/03/13/how-VS-Code-Builds-with-AI)、[Anthropic Engineering — Harness Design for Long-Running Apps](https://www.anthropic.com/engineering/harness-design-long-running-apps)

**这是你们方式 A 的正确形态**。VS Code 团队的具体实现：

> "使用 Playwright MCP 服务器启动 VS Code，导航到待测功能，获取截图，评估变更是否匹配预期行为。由于在 agent 循环中运行，如果截图显示问题，agent 会自动修复。"

**VS Code 的 Golden Scenarios 机制：**
- 录制核心用户流的 golden scenarios（预期行为规范）
- 每次 merge 后自动触发 agent 验证
- 截图存档供人工审查

**Anthropic 三 Agent 分工（前端验收专项）：**

```
Generation Agent  ← 开发前端功能
    ↓（完成后提交给）
Evaluation Agent  ← 用 Playwright MCP 浏览页面、截图、交互
                    按四个维度评分：
                    1. Design Quality（整体一致性）
                    2. Originality（是否有定制化决策）
                    3. Craft（排版/间距/色彩执行）
                    4. Functionality（可用性）
    ↓（评分+批评反馈）
Generation Agent  ← 根据反馈迭代修复（5-15 轮）
```

**关键设计原则（来自 Anthropic）：**
> "Separating the agent doing the work from the agent judging it proves to be a strong lever."

> "Agents tend to respond by confidently praising the work — even when, to a human observer, the quality is obviously mediocre."（自评有系统性偏差，必须独立 Evaluator）

**这正是你们方式 A 效率不够的根因**：让同一个 agent 既开发又验证，会产生自我评价偏差。评估必须由独立 agent 执行。

---

### 方式 4：Visual Regression Testing（组件级快照测试）

来源：[Sauce Labs — 20 Best Visual Regression Testing Tools 2026](https://saucelabs.com/resources/blog/comparing-the-20-best-visual-testing-tools-of-2026)

与 AI 结合后的 smart diffing：AI 区分"有意的设计修改"和"regression bug"，大幅减少误报。

**适用层次：**
- **组件级**（Storybook 集成）：单个组件的像素快照，变更即报警
- **页面级**：整体布局检查，适合上线前回归

**代表工具：** Applitools Eyes、Percy、Chromatic（Storybook 专用）

**局限**：管理 baseline 成本高；按钮移动 2px 就报警会拖慢团队。业界建议只对核心 UI 组件开启。

---

## 二、你们方式 A 效率不足的根本原因

来源：[TestDino](https://testdino.com/blog/playwright-ai-ecosystem/)、[Cegeka](https://www.cegeka.com/en/blogs/agentic-ai-test-automation-with-playwright)

**五个常见失效点：**

| 问题 | 表现 | 解法 |
|------|------|------|
| 幻觉断言 | AI 写出实际不存在的验证 | 必须在 live app 上运行验证，不能只分析截图 |
| 选择器脆弱 | CSS/XPath 一改就断 | 切换到 Accessibility Tree 模式（`getByRole`） |
| 自评偏差 | 同一 agent 开发又验收，总说"没问题" | 独立 Evaluation Agent，物理隔离评判和执行 |
| 测试爆炸 | AI 生成大量重叠场景 | 人工审查 Planner 输出，控制场景边界 |
| 业务逻辑盲区 | Healer 能修选择器，不能修逻辑 | 复杂业务验收仍需人工 AC + 确定性断言 |

---

## 三、Playwright MCP 的两种工作模式

来源：[TestDino — Playwright AI Ecosystem](https://testdino.com/blog/playwright-ai-ecosystem/)

**Snapshot 模式（默认，推荐）：**
- 读取 Accessibility Tree，返回 YAML 结构化数据
- 确定性元素定位，不受 CSS 重构影响
- 速度快，token 消耗低
- **AI agent 首选模式**

**Vision 模式（截图模式，降级选项）：**
- 基于截图做坐标交互
- 用于 Accessibility Tree 读不到的场景（canvas、复杂动画）
- 速度慢，token 消耗高，定位不稳定

**你们现在的问题可能就在这里**：如果用的是 vision 模式（截图分析），切换到 snapshot 模式会显著提升稳定性。

---

## 四、业界的混合策略（生产推荐）

来源：[TestDino](https://testdino.com/blog/playwright-ai-ecosystem/)、[Playwright 官方](https://playwright.dev/docs/test-agents)

> "The smart play is not to make everything agentic."

**按稳定性分层：**

```
Layer 1: 静态确定性测试（不走 AI）
└── 稳定的核心路径（登录、结账、提交）
└── 关键业务逻辑断言
└── 工具：标准 Playwright spec，手动维护

Layer 2: AI 辅助生成 + 人工审查（Planner → Generator）
└── 新功能的测试用例生成
└── 人工审查 Planner 的 Markdown 计划后再生成代码
└── 适合：频繁 UI 变动的功能模块

Layer 3: 自动修复（Healer）
└── 选择器失效的自动修复
└── 超时和等待条件调整
└── 仅限选择器层面，不处理业务逻辑

Layer 4: Evaluation Agent（独立验收）
└── 独立于开发 agent
└── 按 AC 逐条验证
└── 产出结构化报告，驱动 Generation Agent 迭代
```

---

## 五、关于你们的"埋码协议"方案

这个思路方向正确，但不需要自定义协议——业界已有标准答案：

**你们的方案：** 开发时埋自定义 hook → Playwright 共用埋码协议 → 自动化测试

**业界等价方案：** 开发时写正确的 ARIA 属性 → Playwright 用 Accessibility Tree 定位 → 自动化测试

两者本质相同，但业界方案的优势：
1. 不需要维护自定义协议，ARIA 是 W3C 标准
2. 同时满足无障碍访问需求（一举两得）
3. 与 Playwright MCP snapshot 模式天然兼容
4. 所有 AI agent（Claude Code、VS Code Copilot、OpenCode）都能直接理解

**落地做法：**
```jsx
// 不好：自定义埋码
<button data-hook="submit-order">提交</button>

// 好：ARIA 语义（AI 和屏幕阅读器都能理解）
<button aria-label="提交订单" role="button">提交</button>

// Playwright 测试（与 AI agent 高度兼容）
await page.getByRole('button', { name: '提交订单' }).click();
```

---

## 六、实践优先级建议

根据你们当前的痛点（效率和稳定性不够），按投入产出排序：

**第 1 步（最高优先级）：切换到 Snapshot 模式 + Semantic Locator**
- Playwright MCP 从 vision 模式改为 snapshot 模式
- 组件改用 `getByRole` / `getByLabel`，减少 CSS selector 依赖
- 预期效果：稳定性显著提升，token 消耗下降

**第 2 步：独立 Evaluation Agent**
- 验收 agent 和开发 agent 物理隔离
- Evaluation agent 只负责按 AC 验证，不参与开发
- 预期效果：消除自评偏差，验收结果可信

**第 3 步：引入 Playwright 官方 Agents 体系**
- `npx playwright init-agents --loop=opencode`
- 让 Planner 自动生成测试计划，人工审查后 Generator 生成代码
- Healer 接管选择器维护

**第 4 步（可选）：核心组件加 Chromatic/Percy 快照测试**
- 仅对设计系统中的核心组件开启
- 防止像素级 regression，与 CI 集成
