# 前端验收第三条路：落地实施指南

> AI 生成稳定脚本 → 脚本负责执行 → 失败时 AI 自愈  
> 整理时间：2026-04-15

---

## 整体架构

```
┌─────────────────────────────────────────────────────────┐
│  一次性（首次 + 新功能时触发）                           │
│                                                         │
│  验收标准(AC)  →  Planner  →  specs/*.md（人工审查）    │
│                  (AI探索)      ↓                        │
│                            Generator  →  tests/*.spec.ts │
└─────────────────────────────────────────────────────────┘
                                   ↓
┌─────────────────────────────────────────────────────────┐
│  每次 CI（不走 AI，确定性执行）                          │
│                                                         │
│  git push  →  CI  →  npx playwright test（纯脚本执行）  │
│                           ↓                             │
│                   通过 → 绿灯合并                        │
│                   失败 → 触发 Healer                     │
└─────────────────────────────────────────────────────────┘
                                   ↓
┌─────────────────────────────────────────────────────────┐
│  按需（测试失败时）                                      │
│                                                         │
│  失败 trace  →  Healer(AI)  →  修复 selector/断言       │
│                              →  PR 提交人工审查          │
└─────────────────────────────────────────────────────────┘
```

**关键点**：AI 只在"生成"和"修复"阶段介入，CI 执行阶段**不消耗任何 token**。

---

## 前提：先修复现有方案 A 的速度问题（5 分钟）

你们现有的 `opencode.json` 里 playwright MCP 配置：

```json
"playwright": {
  "type": "local",
  "command": ["npx", "-y", "@playwright/mcp@latest", "--isolated"],
  "enabled": true,
  "timeout": 30000
}
```

**确认是否在用 snapshot 模式**（默认应该是，但需要验证）：

```bash
# 在 opencode 对话里发这条消息，看返回的是 YAML 结构还是截图描述
使用 playwright 打开 https://example.com，告诉我页面的 accessibility tree 结构
```

如果返回的是类似这样的 YAML 结构，说明 snapshot 模式正常：
```yaml
- role: main
  children:
    - role: heading
      name: "Example Domain"
    - role: paragraph
      children:
        - role: link
          name: "More information..."
```

如果返回的是截图描述，说明在用 vision 模式，性能会低 10–100x。加 `--no-vision` 参数强制使用 snapshot：

```json
"command": ["npx", "-y", "@playwright/mcp@latest", "--isolated", "--no-vision"]
```

> 来源：[@playwright/mcp v0.0.67+ 默认已启用增量 snapshot，无需额外配置](https://testdino.com/blog/playwright-mcp/)

---

## 第一阶段：初始化 Playwright Agents（30 分钟）

### 1.1 安装依赖

```bash
cd 你的前端项目根目录
npm install -D @playwright/test@latest
npx playwright install
```

### 1.2 初始化 Agents（选择 opencode loop）

```bash
npx playwright init-agents --loop=opencode
```

生成的目录结构：

```
your-project/
├── .github/
│   └── agents/              ← agent 定义文件（Planner/Generator/Healer 的指令）
├── specs/                   ← 放人工可读的 Markdown 测试计划
│   └── example.md           ← 示例
├── tests/
│   └── seed.spec.ts         ← 种子测试（你要填写的基础）
├── playwright.config.ts
└── .vscode/
    └── mcp.json             ← VS Code MCP 配置（已自动生成）
```

> **重要**：每次升级 Playwright 版本后重跑这条命令，更新 agent 定义。

---

## 第二阶段：编写 Seed Test（最重要的一步）

Seed test 是 Planner 的"起点"——它告诉 Planner 如何初始化环境、登录、导航到待测页面。

**Seed test 质量直接决定后续所有生成测试的质量。**

### 2.1 基础结构

```typescript
// tests/seed.spec.ts
import { test, expect } from '@playwright/test';

test('seed', async ({ page }) => {
  // 1. 导航到应用
  await page.goto('http://localhost:3000');

  // 2. 登录（如果需要）
  await page.getByLabel('用户名').fill('test@example.com');
  await page.getByLabel('密码').fill('testpassword');
  await page.getByRole('button', { name: '登录' }).click();

  // 3. 等待登录成功
  await expect(page.getByRole('heading', { name: '首页' })).toBeVisible();
});
```

### 2.2 带认证状态持久化（推荐）

避免每次测试都重新登录，大幅提速：

```typescript
// tests/auth.setup.ts
import { test as setup } from '@playwright/test';
import path from 'path';

const authFile = path.join(__dirname, '../.auth/user.json');

setup('authenticate', async ({ page }) => {
  await page.goto('http://localhost:3000/login');
  await page.getByLabel('邮箱').fill('test@example.com');
  await page.getByLabel('密码').fill('testpassword');
  await page.getByRole('button', { name: '登录' }).click();
  await page.waitForURL('**/dashboard');

  // 保存认证状态
  await page.context().storageState({ path: authFile });
});
```

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  projects: [
    {
      name: 'setup',
      testMatch: /auth\.setup\.ts/,
    },
    {
      name: 'chromium',
      use: {
        storageState: '.auth/user.json',  // 直接复用登录状态
      },
      dependencies: ['setup'],
    },
  ],
});
```

### 2.3 Seed test 写作原则

**用语义选择器，不用 CSS/XPath：**
```typescript
// ❌ 脆弱，UI 改了就断
await page.locator('#btn-submit').click();
await page.locator('.form-input[name="email"]').fill('test@test.com');

// ✅ 稳定，AI 和 Accessibility Tree 都能理解
await page.getByRole('button', { name: '提交' }).click();
await page.getByLabel('邮箱').fill('test@test.com');
await page.getByText('确认订单').click();
```

**断言要明确：**
```typescript
// ❌ 太宽泛
await expect(page).toHaveURL(/dashboard/);

// ✅ 精确，AI 能理解预期状态
await expect(page.getByRole('heading', { name: '订单已提交' })).toBeVisible();
await expect(page.getByText('订单编号：')).toBeVisible();
```

---

## 第三阶段：让 Planner 生成测试计划

### 3.1 在 opencode 里触发 Planner

打开你的前端项目，在 opencode 里发以下消息（以电商结账功能为例）：

```
请作为 Playwright Test Planner，为我们的结账功能生成测试计划。

功能描述：
- 用户可以查看购物车
- 填写收货地址
- 选择支付方式（信用卡/支付宝）
- 提交订单
- 查看订单确认页

验收标准：
- 地址字段必填验证
- 库存不足时显示提示
- 支付成功后跳转到订单确认页
- 订单号显示在确认页

请使用 seed test（tests/seed.spec.ts）作为基础，探索应用并生成 specs/checkout.md
```

### 3.2 Planner 输出的 specs 格式示例

Planner 会在 `specs/` 目录生成类似这样的 Markdown：

```markdown
# 结账功能测试计划

## 应用概述
电商平台结账流程，从购物车到订单确认。

## 测试场景

### 场景 1：正常结账流程
**步骤：**
1. 导航到购物车页面
2. 确认商品列表显示正确
3. 点击"去结账"按钮
4. 填写收货地址（姓名、电话、地址）
5. 选择支付方式：信用卡
6. 填写卡号信息
7. 点击"提交订单"
8. 验证跳转到订单确认页
9. 验证订单号显示

**预期结果：** 订单成功创建，显示订单编号

---

### 场景 2：地址字段验证
**步骤：**
1. 进入结账页面
2. 不填写任何地址信息
3. 点击"提交订单"

**预期结果：** 显示地址必填错误提示

---

### 场景 3：库存不足提示
（...以此类推）
```

**关键步骤：人工审查这个 Markdown，补充遗漏场景，删除不必要的场景。**

---

## 第四阶段：Generator 生成可执行脚本

### 4.1 触发 Generator

在 opencode 里：

```
请作为 Playwright Test Generator，基于 specs/checkout.md 生成可执行的测试脚本。

要求：
- 使用 getByRole/getByLabel/getByText 等语义选择器
- 断言要精确
- 输出到 tests/checkout/ 目录
- 每个场景一个独立文件
```

### 4.2 Generator 生成的脚本示例

```typescript
// tests/checkout/normal-checkout.spec.ts
import { test, expect } from '../fixtures';

test('正常结账流程', async ({ page }) => {
  // 导航到购物车
  await page.goto('/cart');
  await expect(page.getByRole('heading', { name: '购物车' })).toBeVisible();

  // 进入结账
  await page.getByRole('button', { name: '去结账' }).click();
  await expect(page.getByRole('heading', { name: '填写收货信息' })).toBeVisible();

  // 填写地址
  await page.getByLabel('收货人姓名').fill('张三');
  await page.getByLabel('手机号').fill('13800138000');
  await page.getByLabel('详细地址').fill('北京市朝阳区xxx街道');

  // 选择支付方式
  await page.getByRole('radio', { name: '信用卡' }).check();

  // 提交订单
  await page.getByRole('button', { name: '提交订单' }).click();

  // 验证结果
  await expect(page.getByRole('heading', { name: '订单已提交' })).toBeVisible();
  await expect(page.getByText(/订单编号：\d+/)).toBeVisible();
});
```

```typescript
// tests/checkout/address-validation.spec.ts
import { test, expect } from '../fixtures';

test('地址字段必填验证', async ({ page }) => {
  await page.goto('/checkout');

  // 不填地址直接提交
  await page.getByRole('button', { name: '提交订单' }).click();

  // 验证错误提示
  await expect(page.getByText('收货人姓名不能为空')).toBeVisible();
  await expect(page.getByText('手机号不能为空')).toBeVisible();
});
```

### 4.3 运行验证生成的脚本

```bash
# 先在本地跑一遍验证生成的脚本是否正确
npx playwright test tests/checkout/ --headed

# 如果某个测试挂了，看 trace
npx playwright show-trace test-results/checkout-normal-checkout-trace.zip
```

---

## 第五阶段：CI 集成（核心，完全不走 AI）

### 5.1 GitHub Actions 配置

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium

      - name: Start dev server
        run: npm run dev &
        # 或者 start: npx serve dist

      - name: Wait for server
        run: npx wait-on http://localhost:3000

      - name: Run Playwright tests
        run: npx playwright test
        # 纯脚本执行，零 AI 调用，零 token 消耗

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7
```

### 5.2 playwright.config.ts 生产配置

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  
  // 失败时重试一次（给 Healer 触发机会）
  retries: process.env.CI ? 1 : 0,
  
  // 并行执行，加速
  workers: process.env.CI ? 4 : undefined,

  // 失败时保存 trace（Healer 需要用）
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',   // 只在重试时录制 trace
    screenshot: 'only-on-failure',
    video: 'on-first-retry',
  },

  reporter: [
    ['html'],
    ['json', { outputFile: 'test-results/results.json' }],
  ],
});
```

---

## 第六阶段：Healer 自愈配置

### 6.1 手动触发 Healer

当 CI 测试失败后，在 opencode 里：

```
请作为 Playwright Test Healer，修复以下失败的测试：

失败测试文件：tests/checkout/normal-checkout.spec.ts
错误信息：
  Error: locator.click: Error: strict mode violation: getByRole('button', { name: '提交订单' }) 
  resolved to 2 elements.

请查看当前应用状态，找到正确的选择器，修复测试并说明修改原因。
```

### 6.2 自动化 Healer 触发（可选）

在 CI 失败时自动提 PR 让 Healer 修复：

```yaml
# .github/workflows/e2e.yml 里追加
      - name: Trigger Healer on failure
        if: failure()
        run: |
          # 把失败信息写入文件，供 Healer 读取
          npx playwright test --reporter=json 2>&1 | tee test-results/failures.json
          echo "Tests failed. Healer should be triggered manually in opencode."
          echo "Run: cat test-results/failures.json and provide to opencode Healer"
```

### 6.3 Healer 的能力边界

**Healer 能自动修复的：**
- Selector 失效（UI 改了元素路径）—— **75%+ 成功率**
- 超时问题（等待时间不够）
- 断言文本微调（"提交" 改成了 "确认提交"）

**Healer 修不了的（需要人工处理）：**
- 业务逻辑变更（原来成功现在要验证码）
- API 返回格式变化
- 新增的必填字段
- 真实的功能 bug

---

## 可选增强：ZeroStep（复杂交互场景）

来源：[GitHub zerostep-ai/zerostep](https://github.com/zerostep-ai/zerostep)

对于 Generator 生成的脚本里难以用语义选择器定位的复杂场景，可以在脚本里嵌入 `ai()` 步骤：

```bash
npm i @zerostep/playwright -D
export ZEROSTEP_TOKEN="your-token"
```

```typescript
// tests/checkout/complex-payment.spec.ts
import { test, expect } from '@playwright/test';
import { ai } from '@zerostep/playwright';

test('第三方支付流程', async ({ page }) => {
  await page.goto('/checkout');

  // 标准步骤用确定性脚本（快）
  await page.getByRole('radio', { name: '支付宝' }).check();
  await page.getByRole('button', { name: '提交订单' }).click();

  // 第三方页面用 ai()（灵活）
  await ai('在支付宝页面输入支付密码 123456 并确认', { page, test });

  // 回到自己页面后恢复确定性脚本
  await page.waitForURL('**/order-success');
  await expect(page.getByRole('heading', { name: '支付成功' })).toBeVisible();
});
```

**原则**：`ai()` 只用在"无法预知 DOM 结构"的第三方页面或复杂动态交互，其余全用确定性选择器。

---

## 日常工作流

### 新功能开发时

```
1. 开发写 AC（验收标准）
2. 在 opencode 触发 Planner → 生成 specs/feature-name.md
3. 人工审查 specs，补充边界场景
4. 在 opencode 触发 Generator → 生成 tests/feature-name/*.spec.ts
5. 本地跑脚本验证
6. 提交 PR，CI 自动跑脚本（不走 AI）
```

### UI 改版时

```
1. 前端改 UI
2. CI 跑测试 → 部分测试失败（selector 失效）
3. 在 opencode 触发 Healer，提供失败 trace
4. Healer 修复 selector，提 PR
5. 人工审查 Healer 的修改，合并
```

### 业务逻辑变更时

```
1. 修改对应 specs/*.md（更新验收标准）
2. 在 opencode 触发 Generator 重新生成相关测试
3. 验证后合并
```

---

## 成本对比（基于实测数据）

来源：[Bug0 — Playwright MCP Build vs Buy](https://bug0.com/blog/playwright-mcp-changes-ai-testing-2026)、[Playwright Agents 实测](https://medium.com/@twinklejjoshi/playwright-agents-the-future-of-intelligent-test-automation-3d2445fcb1c9)

| 维度 | 你们现在（方案 A） | 第三条路 |
|------|----------------|---------|
| 每次 CI 验收 token 消耗 | ~114K tokens | **0 tokens**（纯脚本） |
| 每次 CI 执行速度 | 分钟级（AI 探索） | 秒级（脚本执行） |
| 测试创建时间 | 每次手动触发 | 比手动快 **70–80%** |
| 测试维护时间 | 每次 UI 变化手动触发 | 减少 **60–70%**（Healer 自愈） |
| 前端开发额外工作 | 零 | **零**（不需要埋码） |

---

## 常见问题

**Q：Planner 生成的计划场景太多怎么办？**  
A：人工审查时删掉重复和低价值场景，只保留核心路径 + 关键边界条件。AI 生成倾向于"穷举"，需要人工剪枝。

**Q：生成的脚本 selector 不稳定怎么办？**  
A：检查前端组件是否有正确的 ARIA 属性（`aria-label`、`role`）。Generator 的输出质量取决于 Accessibility Tree 的语义质量。组件加 ARIA 属性是一次性投入，长期受益。

**Q：测试需要数据库数据怎么处理？**  
A：在 seed test 里做数据准备，或者用 `beforeEach` 通过 API 创建测试数据，用 `afterEach` 清理。

**Q：CI 里跑测试需要启动整个前端服务吗？**  
A：可以用 `webServer` 配置让 Playwright 自动启动：  
```typescript
// playwright.config.ts
webServer: {
  command: 'npm run dev',
  url: 'http://localhost:3000',
  reuseExistingServer: !process.env.CI,
}
```

**Q：Healer 修复的内容可信吗？**  
A：Healer 只修 selector 和等待条件，不修业务逻辑断言。需要人工 review PR 再合并，不要设置自动合并。
