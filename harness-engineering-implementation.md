# Harness Engineering 业界落地实践全览

> 本文综合 15+ 篇一手来源，专注于**具体实现**，不讲概念。每个实践标注出处。
> 整理时间：2026-04-14

---

## 一、核心认知转变：从"让模型做对"到"让错误不可能发生"

来源：[Harness Engineering: Building the Operating System for Autonomous Agents — Medium/AI Forum](https://medium.com/the-ai-forum/harness-engineering-building-the-operating-system-for-autonomous-agents-1e20c105f689)

> "Old mindset: How do I make the model do the right thing?  
> Harness mindset: How do I architect a system where the wrong thing is impossible?"

这不是哲学问题，是工程问题。自动保险理赔的 POC 数据对比：

| 测试场景 | 无 Harness | 有 Harness |
|---------|-----------|-----------|
| 普通碰撞理赔 | 3/4 正确 | 4/4 正确 |
| 高价值理赔（$24.5K） | 2/4（违规审批） | 4/4 |
| 可疑理赔（无目击者） | 2/4 | 4/4 |
| 保单已过期 | 1/4（灾难性失误） | 4/4 |

**没有 harness 的模型得分 50%，有 harness 的得 100%**——正确性被架构保证，不依赖模型训练。

> "The model literally cannot return an approved decision for a lapsed policy because the harness intercepts and rejects any response that tries."

---

## 二、七大落地组件详解

### 2.1 AGENTS.md / CLAUDE.md：持久化记忆

来源：[HumanLayer — Skill Issue: Harness Engineering for Coding Agents](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents)、[Medium — From Prompt Engineering to Harness Engineering](https://medium.com/@cenrunzhe/from-prompt-engineering-to-harness-engineering-the-layer-that-makes-ai-agents-actually-work-466fe0489fbe)

**核心原则：Less is more**

- 文件建议 **不超过 60–100 行**
- ETH Zurich 研究：自动生成的 AGENTS.md **损害性能 20%+**；精心手写的提升 **约 4%**
- 只写"为什么不能这么做"，不写"应该怎么做"（后者太长，前者更精准）

**内容结构建议（来源：NxCode 实操指南）：**
```markdown
# AGENTS.md

## 架构规则（绝对禁止）
- 禁止跨层直接调用：UI 不得直接调用 DB
- 禁止在 Service 层做 HTTP 请求

## 代码规范（已验证失败的模式）
- 不要用 any 类型（曾导致生产 bug，见 PR #234）
- 日志必须用结构化格式

## 测试要求
- 所有新函数必须有单元测试
- 集成测试跑 `npm run test:e2e`

## 已知 agent 失误记录
- 不要自动删除 node_modules（曾造成构建失败）
```

**关键洞察（来源：[Mitchell Hashimoto — My AI Adoption Journey](https://mitchellh.com/writing/my-ai-adoption-journey)）：**
> 每一行都是一个真实的 agent 失误。每次 agent 犯错，就把修复方案写进文件，确保它永远不再犯同样的错。

---

### 2.2 工具系统设计：少即是多

来源：[HumanLayer](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents)、[Claude Code Harness Architecture — WaveSpeed AI](https://wavespeed.ai/blog/posts/claude-code-agent-harness-architecture/)

**反直觉结论：工具越多，性能越差**

Vercel 的实践：移除 80% 的 agent 工具，结果反而更好。原因：工具描述被注入 system prompt，工具越多上下文窗口越挤，agent 越早进入"dumb zone"。

**工具设计原则：**
1. 每个工具描述要精确，避免模糊——工具描述是 prompt injection 的攻击面
2. MCP 服务器建议上限 **5–6 个**（subprocess 开销）
3. 工具按需发现：先加载名称，相关 schema 按需获取，不一次全塞进上下文

**Claude Code 的三层权限模型（来源：WaveSpeed AI）：**
- **Tier 1（自动审批）**：只读操作——文件读、代码导航
- **Tier 2（询问确认）**：状态修改——文件编辑、shell 命令。用后台分类器评估，刻意排除模型的 prose 防止 prompt injection
- **Tier 3（显式批准/拒绝）**：高风险操作——不可预测的系统修改、数据泄露尝试

> "Deny → Ask → Allow（deny 永远赢）"

---

### 2.3 反馈循环：让 agent 验证自己的工作

来源：[Decoding AI — Agentic Harness Engineering](https://www.decodingai.com/p/agentic-harness-engineering)、[LangChain — The Anatomy of an Agent Harness](https://blog.langchain.com/the-anatomy-of-an-agent-harness/)

**核心数据：**
> "Feedback loops improve quality by two to three times"

**三种反馈机制：**

**① 确定性反馈（毫秒级）**：测试、linter、类型检查
```bash
# hooks 配置示例：每次文件修改后自动触发
"post_edit": "npm run typecheck && npm run lint"
```
关键原则：**成功无声，只有失败才向 agent 报告**——冗余输出会浪费上下文。

**② AI 推理反馈（秒级）**：code review agent、"LLM as judge"
- Anthropic 的做法：用独立的 Evaluation Agent 评估 Generation Agent 的产出
  > "Separating the agent doing the work from the agent judging it proves to be a strong lever."（来源：[InfoQ — Anthropic Three-Agent Harness](https://www.infoq.com/news/2026/04/anthropic-three-agent-harness-ai/)）

**③ 自验证循环（"Ralph Loop"）**：agent 写完代码 → 运行测试 → 看日志 → 修复 → 再运行
- 沙盒环境提供：语言运行时、git、浏览器——让 agent 在隔离环境里完整跑一遍

---

### 2.4 上下文管理：对抗"Context Rot"

来源：[Warmly.ai — 9 AI Agents in Production](https://www.warmly.ai/p/blog/agent-harness-for-gtm)、[WaveSpeed AI — Claude Code Architecture](https://wavespeed.ai/blog/posts/claude-code-agent-harness-architecture/)

**关键数据（Warmly 生产环境）：**
> "Models effectively utilize only 8K-50K tokens regardless of context window size."  
> "Information buried in long context shows 20% performance degradation."

**五种应对策略：**

**① 工具输出截断**
- Claude Code 默认限制工具输出 **25K tokens**，警告阈值 **10K**
- 大结果持久化到磁盘，不放入上下文

**② 自动压缩（Auto-compaction）**
- 上下文窗口到达 **98%** 时触发：早期历史摘要化，关键元数据保留
- 重要信息存入 CLAUDE.md，**每轮重新读取**

**③ 渐进式技能加载（Progressive Skill Disclosure）**
- 技能按需加载，不一次性全塞进 instruction budget
- 技能目录可包含多个 markdown 文件，灵活组合

**④ 虚拟文件系统（Virtual Filesystem）**
- 每一步的中间状态写入持久存储
- agent 可以重置上下文窗口后继续工作，不丢失进度

**⑤ 子 agent 隔离（Context Firewall）**
- 子 agent 在独立上下文窗口执行，完成后只汇报结果
- 防止中间过程污染父 agent 的上下文

---

### 2.5 中间件栈：每次模型调用都经过的四道门

来源：[Medium/AI Forum — Building the Operating System](https://medium.com/the-ai-forum/harness-engineering-building-the-operating-system-for-autonomous-agents-1e20c105f689)

每次调用模型都要经过：

```
Model Call
    ↓
① Observability Middleware    ← 记录所有调用
    ↓
② Budget Enforcement          ← token/调用次数限制
    ↓
③ Loop Detection              ← 停止重复模式
    ↓
④ Domain Rule Verification    ← 拒绝违规响应
    ↓
Execute / Return
```

这四道门**纯代码实现**，不依赖模型理解规则：
> "The model literally cannot return an approved decision for a lapsed policy because the harness intercepts and rejects any response that tries."

---

### 2.6 多 Agent 协调：防碰撞与全局一致性

来源：[Warmly.ai](https://www.warmly.ai/p/blog/agent-harness-for-gtm)、[Anthropic Three-Agent Harness — InfoQ](https://www.infoq.com/news/2026/04/anthropic-three-agent-harness-ai/)

**核心问题：Agent Collision**

多个 agent 各自做局部最优决策，导致全局糟糕结果。Warmly 的真实案例：两个 agent 在几小时内给同一个潜在客户发了两封邮件。

**Warmly 的四层 Harness 架构（9 个生产 agent 共用）：**

```
Layer 1: Context Graph        ← 统一数据源（公司信息、网站活动、CRM）
                                  消除 API 竞争条件和过期缓存
Layer 2: Policy Engine        ← YAML 规则，集中定义 agent 能/不能做什么
Layer 3: Decision Ledger      ← 完整决策日志（上下文、策略、置信度、行动）
Layer 4: Outcome Loop         ← 决策→业务结果的闭环（会议预约、交易关闭）
```

**防碰撞：基于 Temporal 的事件路由**
```yaml
# Policy Engine YAML 规则示例
rules:
  - max_touches_per_day: 1
  - channel_cooldown_hours: 72
  - same_channel_min_interval_hours: 168
```
每个 agent 的行动都发布到共享事件流，路由层执行规则。

**Anthropic 的三 Agent 分工（来源：InfoQ）：**
```
Planning Agent      ← 建立任务结构和规格（JSON feature specs）
    ↓
Generation Agent    ← 执行实际开发工作
    ↓
Evaluation Agent    ← 独立评估产出（用 Playwright MCP 与实际界面交互）
```
关键设计：context reset 而非 compaction——通过结构化 handoff artifacts 实现无缝切换，避免压缩导致的模型行为保守化。每次运行迭代 **5–15 次**，有时超过 **4 小时**。

---

### 2.7 Hooks：生命周期自动化

来源：[HumanLayer](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents)、[Claude Code Harness — GitHub](https://github.com/Chachamaru127/claude-code-harness)

Hooks 在生命周期事件自动触发，不需要 agent 主动调用。

**Claude Code Harness v4 的 13 条 Guardrail 规则（Go 实现，sub-10ms）：**

| 规则 ID | 防护对象 | 触发动作 |
|---------|---------|---------|
| R01–R02 | `sudo`、`.git/`、`.env`、密钥 | **拒绝** |
| R03–R06 | 项目外写入、`git push --force` | **拒绝 / 询问** |
| R10–R11 | `--no-verify`、`git reset --hard main` | **拒绝** |
| R12–R13 | 直接推 main、修改受保护文件 | **警告** |

**Plan→Work→Review 自动化循环（来源：GitHub Chachamaru127/claude-code-harness）：**
```
/harness-setup  → 初始化项目配置
/harness-plan   → 想法 → Plans.md（含验收标准）
/harness-work   → 并行实现 + 预检自查 + 独立评审
/harness-review → 4 视角代码评审（安全/性能/质量/可访问性）
/harness-release → CHANGELOG + tag + GitHub release
```
`/harness-work all` 可把整个流程合并成一个命令：批准计划 → 并行实现 → 评审 → 提交。

---

## 三、三级落地路径

来源：[NxCode — Harness Engineering Complete Guide](https://www.nxcode.io/resources/news/harness-engineering-complete-guide-ai-agent-codex-2026)

| 级别 | 适用场景 | 搭建时间 | 核心工具 |
|------|---------|---------|---------|
| **Basic** | 个人开发者 | 1–2 小时 | `.cursorrules`、pre-commit hooks、测试套件 |
| **Team** | 3–10 人团队 | 1–2 天 | `AGENTS.md`、CI 约束、review checklist |
| **Production** | 组织级 | 1–2 周 | 自定义中间件、可观测性平台、监控 dashboard |

**Warmly 的成本参考（9 个生产 agent）：**
- 自建第一年：$25–50 万
- 后续年度维护：$15–30 万 + 1–2 名高级工程师
- 最小可行 harness：**4 周**（统一上下文 + 事件流 + 决策日志 + 结果追踪）

---

## 四、Red Hat 的结构化任务模板

来源：[Red Hat Developer — Harness engineering: Structured workflows](https://developers.redhat.com/articles/2026/04/07/harness-engineering-structured-workflows-ai-assisted-development)

Red Hat 把 harness 落地成一套两阶段工作流：

**Phase 1：Repository Impact Map**
- AI 用 LSP + MCP 扫描真实代码库，产出受影响文件清单
- **关键检查点：人工审核**，在任务创建前发现结构性错误

**Phase 2：结构化任务模板**
```
每个任务必须包含：
- Repository：单一代码库范围
- Files to Modify：真实路径（不是猜测）
- Implementation Notes：引用实际符号和现有模式
- Acceptance Criteria：具体可验证的检查清单
- Test Requirements：特定覆盖率指引
```

> "The more you constrain the solution space, the more predictable the output becomes."

> "The AI writes better code when you design the environment it works in."

---

## 五、arXiv 论文中的五层安全纵深

来源：[Building AI Coding Agents for the Terminal — arXiv 2603.05344](https://arxiv.org/html/2603.05344v1)

生产级 agent 的安全不依赖单一机制，而是五层独立拦截：

```
Layer 1: Prompt Guardrails        ← 系统提示级别的行为约束
Layer 2: Schema Restrictions      ← 工具 schema 层面禁止危险参数
Layer 3: Runtime Approvals        ← 运行时人工审批高风险操作
Layer 4: Tool-Level Validation    ← 每个工具自己的校验逻辑
Layer 5: Lifecycle Hooks          ← 前后钩子兜底
```

任何一层都不依赖其他层，无单点故障。

**双模式设计（Plan/Normal）在 schema 层强制分离：**
- **Plan Mode**：只读工具 + Planner 子 agent 做代码库探索
- **Normal Mode**：完整工具访问，执行已批准计划

不用运行时状态机，用 schema 本身保证安全边界。

---

## 六、LangChain 对 Harness 解剖

来源：[LangChain Blog — The Anatomy of an Agent Harness](https://blog.langchain.com/the-anatomy-of-an-agent-harness/)

六个核心子系统：

| 子系统 | 功能 | 关键设计 |
|--------|------|---------|
| **Storage & State** | 跨 session 持久化 | 文件系统作为基础原语，多 agent 协作面 |
| **Code Execution** | 通用问题解决 | Bash 让 agent 写代码并自己执行 |
| **Safe Execution Env** | 隔离沙盒 | 语言运行时 + git + 浏览器，支持自验证循环 |
| **Memory & Knowledge** | 跨截止日期信息 | 记忆文件 + web search + MCP 工具 |
| **Context Management** | 对抗 context rot | 压缩、工具输出卸载、渐进技能加载 |
| **Long-Horizon Exec** | 复杂多步任务 | 规划、验证循环、Ralph Loop |

> "As models improve, harness sophistication will shift from patching deficiencies toward optimizing systems around model capabilities."

---

## 七、Cobus Greyling 的框架层演变理论

来源：[Cobus Greyling — The Rise of AI Harness Engineering](https://cobusgreyling.medium.com/the-rise-of-ai-harness-engineering-5f5220de393e)

**传统框架正在分解：**
- 约 **80%** 的传统框架职责（agent 定义、路由、编排）已被强模型原生吸收
- 剩余 **20%**（持久化、成本控制、可观测性、错误恢复）才是 harness 的真正领域

**三个生产级案例：**
1. **Claude Code**：管理代码库、文件系统、子 agent、跨 session 记忆
2. **OpenAI Codex**：100 万行代码，"no manually typed code at all"，harness 是第一优先级
3. **OpenAI CUA Sample App**：管理截图-行动-验证循环

---

## 八、SmartScope 的四象限 Harness 模型

来源：[SmartScope — What Is Harness Engineering: A New Concept](https://smartscope.blog/en/blog/harness-engineering-overview/)

```
┌─────────────────────┬─────────────────────┐
│  Architecture       │  Feedback           │
│  Constraints        │  Loops              │
│  (Linter 机械执行)  │  (CI/CD 验证)       │
├─────────────────────┼─────────────────────┤
│  Workflow           │  Improvement        │
│  Control            │  Cycles             │
│  (任务拆分/权限)    │  (熵管理/文档更新)  │
└─────────────────────┴─────────────────────┘
```

量化数据：
- Can.ac 实验：只改 harness 工具格式，模型从 **6.7% → 68.3%**（10 倍提升）
- LangChain 实验：harness 优化后排名从 **第 30 → 第 5**

诊断方法：
> "If quality isn't improving despite better prompts, distinguish whether the problem lies in context or harness layers."

---

## 九、Warmly 生产 9 个 Agent 的核心教训

来源：[Warmly.ai — The Agent Harness: What We Learned Running 9 AI Agents in Production](https://www.warmly.ai/p/blog/agent-harness-for-gtm)

**工具调用在生产环境的失败率：3–15%**

**80% 的 AI 项目会失败**，40%+ 的 agentic AI 项目预计在 2027 年被取消（成本 + 风险控制不足）。

**四条实战教训：**

1. **统一上下文是地基**：所有 agent 必须从同一个 Context Graph 读数据，消除 API 竞争条件和缓存不一致

2. **策略集中管理**：在 Policy Engine（YAML 规则）里定义行为约束，不要分散在各个 agent 的 prompt 里

3. **决策必须可追溯**：Decision Ledger 记录完整链路——agent 看到了什么、应用了哪条策略、置信度多少、最终做了什么。没有这个，无法 debug，无法向利益相关方解释

4. **闭合结果循环**：把 agent 决策和业务结果（会议预约、交易关闭）挂钩，才能做持续改进

---

## 十、实践起点：从零搭建的优先级顺序

来源综合：[HumanLayer](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents)、[Medium Steven Cen](https://medium.com/@cenrunzhe/from-prompt-engineering-to-harness-engineering-the-layer-that-makes-ai-agents-actually-work-466fe0489fbe)、[Louis Bouchard](https://www.louisbouchard.ai/harness-engineering/)

```
第 1 步（1 小时）
└── 创建 AGENTS.md / CLAUDE.md
    - 60 行以内
    - 只记录真实 agent 失误的修复
    - 每次 agent 犯错就更新

第 2 步（半天）
└── 接入基础 linter + pre-commit hook
    - typecheck 和 lint 自动在 agent 提交前跑
    - 只输出错误，成功不输出

第 3 步（1 天）
└── 工具精简
    - 把 MCP 工具数量降到最低
    - 移除不常用工具，测试 agent 是否反而更好

第 4 步（2–3 天）
└── 子 agent 隔离
    - 把长任务拆分为离散子任务
    - 每个子任务在独立上下文窗口运行

第 5 步（1 周）
└── 反馈循环
    - 加入 Evaluation Agent 独立评估产出
    - 接入 CI 结果，让 agent 能读到测试状态

第 6 步（持续）
└── Decision Ledger + Outcome Loop
    - 记录 agent 的决策链
    - 把决策和真实结果挂钩，建立改进飞轮
```

**最后的原则（来源：HumanLayer）：**
> "Bias toward shipping. Start simple, add configuration only when agents actually fail, and optimize for iteration speed rather than perfect first attempts."
