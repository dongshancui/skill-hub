# Harness Engineering 最新动态调研报告（2026年3–4月）

> 调研截止：2026-04-25  
> 涵盖：GitHub 趋势、arXiv 论文、OpenAI/Anthropic 官方发布、社区工具链、MCP/A2A 协议进展

---

## 一、一句话结论

**"Harness"已成为2026年AI工程最热关键词。**  
模型能力趋于饱和，胜负手转移到 harness——即围绕模型的工具链、约束层、反馈回路、记忆系统的工程体系。  
"Agent = 模型 + Harness"，这个公式正在被整个行业接受。

---

## 二、速览地图（渐进式披露入口）

| 主题 | 核心结论 | 跳转 |
|---|---|---|
| 概念起源 | OpenAI Feb 2026 正式命名，Martin Fowler 背书 | [§3](#三harness-概念的正式确立) |
| Claude Code Harness | 26个生命周期事件，4月新增 mcp_tool handler | [§4](#四claude-code-harness-最新进展) |
| Anthropic 三智能体架构 | Planner→Generator→Evaluator，分离评估比自我批评有效 | [§5](#五anthropic-三智能体-harness-架构) |
| OpenAI 大规模实验 | 3人×5个月，100万行代码，0%人工编写，0%人工审查 | [§6](#六openai-harness-engineering-大规模实验) |
| 论文速览 | AutoHarness/Meta-Harness/AdaptOrch等6篇关键论文 | [§7](#七关键-arxiv-论文) |
| 社区工具链 | everything-claude-code/superpowers/claw-code等生态 | [§8](#八社区工具链生态) |
| 评估工具 | Braintrust $80M融资，六层测试栈成型 | [§9](#九评估工具链现状) |
| MCP/A2A | MCP 2026路线图，Streamable HTTP，A2A v1.0 | [§10](#十mcp--a2a-协议进展) |
| LM Eval Harness | EleutherAI v0.4.11，安全修复，Windows后端 | [§11](#十一lm-evaluation-harness-eleutherai) |
| 核心模式总结 | 内/外harness、PEV架构、Reasoning Sandwich等 | [§12](#十二核心工程模式总结) |

---

## 三、Harness 概念的正式确立

### 3.1 命名时刻

2026年2月11日，OpenAI成员 Ryan Lopopolo 发表博文《Harness engineering: leveraging Codex in an agent-first world》，正式将这一工程实践命名为 **Harness Engineering**。

同期：
- Mitchell Hashimoto（Terraform、Ghostty 作者）将其提炼为：**Agent = Model + Harness**
- Martin Fowler 在 martinfowler.com 发表深度文章《Harness Engineering for Coding Agents》，覆盖从理论到落地的完整体系

### 3.2 为什么"现在"出圈

三个同步发生的催化剂：
1. **OpenAI/Lopopolo 的命名**——给从业者提供了共同语言
2. **Claude Code 源码泄露（3月31日）**——泄露的代码证明 harness 架构即护城河
3. **基准数据**——同一模型，换不同 harness，LangChain 编程智能体在 TerminalBench-2 上从 52.8% 涨到 66.5%

### 3.3 核心定义

> Harness = 围绕模型的**前馈控制（Guides）+ 反馈控制（Sensors）**的工程体系

| 控制类型 | 时机 | 举例 |
|---|---|---|
| 前馈（Feedforward） | 执行前 | 架构文档、编码规范、引导脚本 |
| 反馈（Feedback） | 执行后 | Linter、CI门禁、代码审查智能体 |

执行类型：
- **计算型**（毫秒级）：测试、Linter、类型检查——确定性
- **推理型**（慢、贵）：LLM-as-Judge、代码审查智能体——概率性

---

## 四、Claude Code Harness 最新进展

### 4.1 Hook 系统 v2.1.116（2026年4月）

**26个生命周期事件**可被挂钩，新增 `mcp_tool` handler 类型（4月）：

```
Session:    SessionStart | SessionEnd
Turn:       UserPromptSubmit | Stop | StopFailure
Tool:       PreToolUse | PostToolUse | PostToolUseFailure |
            PermissionRequest | PermissionDenied | PostToolBatch
Async:      Notification | SubagentStart/Stop | TaskCreated/Completed |
            InstructionsLoaded | ConfigChange | CwdChanged | FileChanged |
            WorktreeCreate/Remove | PreCompact/PostCompact |
            Elicitation/ElicitationResult | UserPromptExpansion
```

**Handler 类型（4种→5种）：**

| 类型 | 描述 | 新增 |
|---|---|---|
| `command` | Shell命令，JSON via stdin，exit 2 = 阻断 | — |
| `http` | POST到URL，2xx=成功 | — |
| `prompt` | 让Claude做是/否判断 | — |
| `agent` | 启动子智能体（仅 Read/Grep/Glob 权限） | — |
| `mcp_tool` | 调用MCP服务器上的工具，支持 `${path}` 替换 | **2026-04新增** |

**重要配置字段（近期新增）：**

```json
{
  "if": "...",                      // 条件触发（如仅限 git push）
  "async": true,                    // 非阻塞后台执行
  "asyncRewake": true,              // 后台完成后重新唤醒Claude
  "permissionDecision": "defer",    // 暂停等待外部UI决策
  "CLAUDE_ENV_FILE": "...",         // 跨session持久化环境变量
  "updatedPermissions": {},         // 动态权限规则管理
  "additionalContext": "..."        // 向Claude上下文注入信息
}
```

**PreToolUse 决策输出（安全关键模式）：**
```json
{
  "permissionDecision": "allow|deny|ask|defer",
  "permissionDecisionReason": "原因说明",
  "updatedInput": { "修改的字段": "值" },
  "additionalContext": "追加给Claude的上下文"
}
```
优先级顺序：`deny > defer > ask > allow`

### 4.2 配置作用域层级

```
~/.claude/settings.json              # 全局
.claude/settings.json                # 项目（可提交）
.claude/settings.local.json          # 项目（私有）
Plugin hooks/hooks.json              # 插件级
Skill/Agent frontmatter              # Skill级
Managed policy settings              # 组织级（最高优先级）
```

### 4.3 Claude Code 源码泄露揭示的内部架构（2026-03-31）

3月31日，`@anthropic-ai/claude-code` v2.1.88 npm包意外暴露 59.8MB `.map` 源码（512,000+ 行 TypeScript，1,906个源文件），揭示：

**KAIROS**：常驻后台守护进程
- 持续后台会话，15秒阻断预算
- GitHub webhook订阅，5分钟cron刷新
- **autoDream**：每晚记忆蒸馏（合并观察→消除矛盾→裁剪至≤200行/25KB）
- 目标发布：2026年5月

**其他发现：**
- `anti_distillation` 标志：防止模型被追踪/蒸馏
- Undercover Mode：Anthropic通过Claude Code向公开OSS仓库贡献代码
- 内部模型代号：Capybara（Claude 4.6变体）、Fennec（Opus 4.6）、Numbat（未发布）
- 44个隐藏feature flags

---

## 五、Anthropic 三智能体 Harness 架构

**来源**：Anthropic Engineering Blog《Harness Design for Long-Running Apps》，2026-03-24

### 5.1 三角分工

```
用户 1-4句话需求
        ↓
  ┌─ Planner ─────────────────────┐
  │ 将需求转化为产品规格            │
  │ 识别AI功能机会                 │
  │ 避免写实现细节                 │
  └────────────────────────────────┘
        ↓ 产品规格
  ┌─ Generator ────────────────────┐
  │ 增量实现功能（sprint制）        │
  │ 自评后再移交QA                 │
  │ 技术栈：React/Vite/FastAPI/SQLite│
  └────────────────────────────────┘
        ↓↑ "sprint合约"
  ┌─ Evaluator ────────────────────┐
  │ 像真实用户一样测试              │
  │ 基于标准评分                   │
  │ 工具：Playwright MCP           │
  └────────────────────────────────┘
```

### 5.2 关键发现

1. **分离评估比自我批评有效**：同一智能体的自我批评远不如独立Evaluator
2. **Sprint合约**：Generator和Evaluator在实现前协商可测试的验收标准
3. **成本对比**：
   - 独跑（20分钟，$9）→ 功能少、bug多
   - 三智能体harness（6小时，$200）→ 功能丰富、bug大幅减少
4. Claude Opus 4.6 消除了早期版本的"上下文焦虑"（需要频繁重置）

---

## 六、OpenAI Harness Engineering 大规模实验

**来源**：Ryan Lopopolo + Latent.Space 深度访谈，2026-02

### 6.1 实验规模

| 指标 | 数值 |
|---|---|
| 代码量 | 1M+ 行（生产Electron应用） |
| 时间 | 5个月 |
| 团队 | 3名工程师 |
| PR数量 | 1,500+ |
| Token消耗 | 每日1B+（约$2-3k/天） |
| PR效率 | 3.5 → 5-10 PR/工程师/天 |
| 人工编码比例 | **0%** |
| 合并前人工审查 | **0%**（转为合并后审查） |

### 6.2 核心架构

- **Symphony 编排层**：Elixir（BEAM进程监督）构建；管理worktree、合并冲突解决、CI等待状态；完全自主运行
- **构建系统演进**：Makefile → Bazel → Turbo → NX（目标：<60秒构建周期）
- **6个核心技能**：封装业务逻辑和可观测性

### 6.3 关键工程模式

**Ghost Libraries（幽灵库）**：将软件以规格文档而非源码形式分发

**Spec-Driven Development（规格驱动开发）**：
```
参考代码 → 智能体A生成高保真规格 → 智能体B仅凭规格实现 → 审查智能体验证
```

**代码审查智能体**：合并后触发，优先级P0/P2；智能体倾向于放行，有反推机制防止来回振荡

---

## 七、关键 arXiv 论文

### 7.1 AutoHarness（2603.03329，2026-03）

**核心**：自动为AI智能体合成代码harness  
**关键结论**：
- 小模型 + 自动生成harness > 大模型裸跑
- Gemini-2.5-Flash + harness 在145场 TextArena 棋局中防止所有非法落子
- 78% 的 Gemini-2.5-Flash 失败源于非法动作 → harness = 约束强制执行

工具：`pip install autoharness`，2行代码接入任意OpenAI兼容API

### 7.2 AdaptOrch（2602.16873，2026-02-18）

**核心**：任务自适应多智能体编排  
**关键结论**：
- LLM性能趋于收敛时，**编排拓扑（而非模型选择）决定系统性能**
- 拓扑路由算法：O(|V|+|E|)时间复杂度，映射任务DAG到最优模式（并行/串行/层级/混合）
- 相比静态单拓扑基线提升**12-23%**（模型完全相同）

### 7.3 Natural-Language Agent Harnesses（2603.25723，2026-03-26）

**核心**：将harness设计从控制器代码中解耦，以自然语言表达  
**方案**：
- NLAH（自然语言智能体harness）：可移植、可对比、可研究
- IHR（智能体harness运行时）：通过显式契约、持久化工件、轻量适配器执行harness

### 7.4 Meta-Harness（2603.28052，2026-03-30）

**作者**：Yoonho Lee（Stanford）等  
**核心**：harness本身也应被自动优化，而非手工设计  
**方法**：智能体提议者——有文件系统访问权限，读取源码、得分、历史执行追踪  
**结果**：
- 在线文本分类：超SOTA **+7.7分**，用**少4倍**的上下文token
- RAG数学推理（200道IMO级题目，5个模型平均）：**+4.7分**
- Agentic编程：在TerminalBench-2上超过最优手工基线

### 7.5 The Last Harness You'll Ever Build（2604.21003，2026-04-22）

**核心**：两级自动化体系  
- **Harness Evolution Loop**：单个智能体配置的自动优化（执行→对抗评估→修改）
- **Meta-Evolution Loop**：学习跨任务的最优进化协议 → 新任务上无需人工即可快速收敛

三类智能体角色：Worker（执行）、Evaluator（对抗诊断失败）、Evolution Agent（基于历史修改harness）

### 7.6 Qualixar OS（2604.06392，2026-04-07）

**核心**：AI智能体编排的应用层"操作系统"  
**规模**：10个LLM提供商、8+智能体框架、7种传输协议  
**验证**：2,821个测试用例，任务均价 $0.000039

---

## 八、社区工具链生态

### 8.1 GitHub 热门项目（2026年3-4月）

| 项目 | Stars | 亮点 |
|---|---|---|
| `ultraworkers/claw-code` | 94,240（2周内） | 最快到5万星（2小时）；Claude Code Rust+Python 重写 |
| `affaan-m/everything-claude-code` | 129,198 | 38个智能体、156个技能、72个命令shim；12语言生态 |
| `obra/superpowers` | 129,363 | "强制工作流而非建议工作流"；已入 Anthropic marketplace |
| `NousResearch/hermes-agent` | 65,964 | GEPA自进化；比RL高6-20%、少35倍rollout |
| `coleam00/Archon` | 15,600+ | YAML定义工作流；原始LLM代码6.7% PR接受率→harness后~70% |
| `HKUDS/OpenHarness` | 活跃 | 11,733行Python，LLM无关，复现98%核心工具 |
| `aiming-lab/AutoHarness` | 活跃 | 论文对应实现，6步治理流水线 |

### 8.2 everything-claude-code (ECC) 亮点

**新目录 `ecc2/`**：
- Rust控制平面 + TUI
- AgentShield v1.4.0：CVE数据库（25+已知MCP漏洞）

**跨平台支持**：
- Claude Code（28智能体、60命令、34规则、8个hook事件类型）
- Cursor（15个hook事件类型）
- OpenCode（11个hook事件、6个原生自定义工具）

**安装示例**：
```bash
ecc install --profile developer --with lang:typescript --without skill:continuous-learning
```

### 8.3 Hermes Agent（NousResearch）v0.8.0

**GEPA（遗传进化提示架构）**——ICLR 2026 Oral：
- 读取执行追踪，识别低效（如47次工具调用完成本可12次的任务）
- 提出针对性的提示/技能改进
- 与RL相比：性能高6-20%，rollout少35倍

**v0.8.0新特性**：Browser Use集成、远程后端部署、并行worktree处理、实时模型切换、后台任务自动通知

### 8.4 Archon：让AI编码变确定性

**核心数据**：
- 裸LLM代码 PR接受率：**6.7%**
- 结构化harness后：**~70%**

基于YAML定义17套预设工作流，wraps Claude Code 和 Codex CLI。

### 8.5 claw-code：Claude Code 开源重写

- Rust + Python 实现
- 使用 oh-my-codex（OmX）平行代码审查构建
- 2小时达5万星，一周内94,240星（GitHub有史以来最快）

### 8.6 定价竞争格局（2026年4月）

| 公司 | harness策略 | 价格 |
|---|---|---|
| Anthropic | Managed Agents（托管）+ Claude Agent SDK | $0.08/会话小时 + token费用（2026-04-08发布）|
| OpenAI | 开源 Agents SDK + Codex App Server | 仅token费用（Anthropic发布7天后跟进）|
| Google | ADK（Java 1.0已发布）+ A2A协议 | TBD |

> 共识：harness就是产品。分歧：价格。

---

## 九、评估工具链现状

### 9.1 六层测试栈（2026年行业共识）

```
Layer 6: 认证数据基础（避免训练污染）
Layer 5: 生产CI/CD回归
Layer 4: 对抗性红队测试
Layer 3: 端到端模拟
Layer 2: 集成测试（Braintrust）
Layer 1: 单元测试（DeepEval）
```

### 9.2 主要工具对比

| 工具 | 定位 | 亮点 |
|---|---|---|
| **DeepEval** | 单元/集成测试 | 50+研究指标，pytest原生 |
| **Braintrust** | 全生命周期评估 | $80M B轮（2026-02），$8亿估值；Notion/Replit/Cloudflare等客户 |
| **LangFuse** | 追踪+提示管理 | MIT协议，可自托管 |
| **Promptfoo** | 多供应商prompt回归 | — |
| **RAGAS** | RAG评估 | 忠实度、上下文召回 |
| **AgentOps/Arize Phoenix** | 生产监控 | — |

### 9.3 SWE-bench 现状（2026年4月）

- **SWE-Bench Pro**（新标准）：1,865个任务，41个活跃维护仓库，Python/Go/TS/JS
  - GPT-5 和 Claude Opus 4.1 均约 **23%**
- **SWE-rebench**：Claude Opus 4.6 排名第一
- **SWE-bench-Live**：每月新增50个已验证issue，保持公平性

---

## 十、MCP & A2A 协议进展

### 10.1 MCP 2026 路线图（2026-03发布）

**规模**：10,000+ 活跃MCP服务器；每月9700万次SDK下载

**四大优先方向**：

1. **Transport 演进**：Streamable HTTP 作为无状态/可扩展传输；显式session管理（创建/恢复/迁移）；`.well-known` 能力发现格式
2. **Tasks Primitive**（SEP-1686）：重试语义 + 结果保留过期策略
3. **治理成熟化**：Linux Foundation 所有权；SEP工作组制度
4. **企业就绪**：审计追踪、SSO集成认证、网关行为、配置可移植性

### 10.2 MCP 生产部署现状（2026年4月）

2,181个远程MCP服务器端点分析：
- **52% 完全失效**
- **9% 完全健康**
- 其余：降级或静默失败

企业模式：集中式托管MCP（通用数据源）+ 分布式MCP（区域/敏感系统）

### 10.3 A2A 协议 v1.0

- **Google 2025年4月宣布**，50+合作伙伴
- **v1.0（2026年）**：Signed Agent Cards（加密签名）+ AP2扩展
- 150+组织参与；Linux Foundation 所有权
- 生产部署：Microsoft、AWS、Salesforce、SAP、ServiceNow

### 10.4 Claude Code 新增 mcp_tool Handler

```json
{
  "hooks": {
    "PreToolUse": [{
      "type": "mcp_tool",
      "server": "security-checker",
      "tool": "scan",
      "input": { "path": "${path}" }
    }]
  }
}
```

---

## 十一、LM Evaluation Harness（EleutherAI）

### 11.1 v0.4.11（2026-02）主要变化

- **Windows ML后端**：原生Windows ML推理支持
- **新基准**：BEAR知识探测
- **安全修复**：用 `ast.literal_eval` 替换不安全的 `eval()` 调用
- 任务版本更新：afrobench_belebele、evalita_llm、include（90语言变体）、mgsm_direct

### 11.2 v0.4.10（2026-01）重要变更

- **破坏性变更**：核心包不再默认安装模型后端，需显式指定：
  ```bash
  pip install lm-eval[hf]    # HuggingFace后端
  pip install lm-eval[vllm]  # vLLM后端
  pip install lm-eval[api]   # API后端
  ```
- CLI重构为显式子命令：`lm-eval run`、`lm-eval ls tasks`
- 新增YAML配置文件支持

### 11.3 Hugging Face Open LLM Leaderboard v2

- 使用 lm-evaluation-harness 作为后端引擎
- 基准：IFEval、BBH、MATH、GPQA、MUSR、MMLU-PRO
- 评分归一化：随机=0，满分=100
- 新增：污染检测 + 社区举报机制

---

## 十二、核心工程模式总结

### 12.1 Inner vs. Outer Harness

```
Inner Harness（内置）
  ├── 框架创建者提供
  ├── 系统提示、编排、核心基础设施
  └── 随模型能力提升而缩小

Outer Harness（外加）
  ├── 用户/组织针对具体场景添加
  ├── Linter、CI门禁、约束规则
  └── 确定性约束弥补LLM指令服从的概率性
```

**关键洞见**：LLM对指令的服从是概率性的，不是确定性的——确定性的外部harness约束（Linter、CI门禁）是生产可靠性的必要条件。

### 12.2 Plan-Execute-Verify（PEV）架构

> 每步缺乏验证 → 即使单步准确率高，错误也会指数级累积 → 系统级失败

```
Plan → Execute → Verify → Plan（下一步）→ ...
```

### 12.3 Reasoning Sandwich 模式

```
高成本高能力模型（规划 + 验证）
        ↕
快速廉价模型（中间数据收集 + 工具执行）
```

### 12.4 上下文管理关键实践

**Prefix Stability（前缀稳定性）**：
- 系统指令/历史保持静态 → KV缓存命中 → 成本从 $3.00/MTok → $0.30/MTok

**Progressive Disclosure（渐进式注入）**：
- 工具文档/schema 按需注入，而非全量加载

**Virtual Memory 模式**：
- 上下文外化到 `AGENTS.md`、`todo.md` 日志
- 计划作为版本化的一等公民工件

### 12.5 三类 Harness 功能分层

| 层次 | 目标 | 工具 |
|---|---|---|
| 可维护性 Harness | 重复代码、圈复杂度、覆盖率 | Linter、测试覆盖率门禁 |
| 架构适配度 Harness | 依赖流向、可观测性规范、性能SLO | 架构测试、依赖分析 |
| 行为 Harness | 功能正确性（最难） | 变异测试、AI生成测试套件（仍不充分） |

### 12.6 变更生命周期中的时机选择

```
pre-commit:        快速Linter、基础代码审查智能体
pre-integration:   快速测试套件、类型检查
post-integration:  变异测试、全面审查（昂贵）
continuous:        漂移检测、SLO运行时监控
```

---

## 十三、关注指标与风险

### 企业采用现状

- **88% 企业AI智能体项目**无法进入生产
- **65%的失败**源于"Harness缺陷"（上下文漂移、Schema错位、状态退化），而非模型能力
- Gartner：2026年底40%企业将采用任务特定AI智能体；但>40%的agentic AI项目将在2027年前因评估/数据质量不足而取消
- **80% 的agentic AI实施时间**消耗在数据工程和治理上

### 经济杠杆

- **语义缓存**：预计算重复评估
- **Logits Masking**：工具访问权限按任务阶段动态缩放
- **Ralph Loop Recovery**：上下文提前退出后，向新的干净上下文窗口重新注入原始意图

---

## 十四、参考来源

**官方发布：**
- [OpenAI — Harness Engineering](https://openai.com/index/harness-engineering/)
- [OpenAI — Codex App Server](https://openai.com/index/unlocking-the-codex-harness/)
- [Anthropic Engineering — Harness Design for Long-Running Apps](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- [Martin Fowler — Harness Engineering for Coding Agents](https://martinfowler.com/articles/harness-engineering.html)
- [MCP 2026 Roadmap](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/)

**深度访谈：**
- [Ryan Lopopolo — Latent.Space](https://www.latent.space/p/harness-eng)

**arXiv 论文：**
- [AutoHarness — 2603.03329](https://arxiv.org/abs/2603.03329)
- [AdaptOrch — 2602.16873](https://arxiv.org/abs/2602.16873)
- [Natural-Language Agent Harnesses — 2603.25723](https://arxiv.org/abs/2603.25723)
- [Meta-Harness — 2603.28052](https://arxiv.org/abs/2603.28052)
- [The Last Harness You'll Ever Build — 2604.21003](https://arxiv.org/abs/2604.21003)
- [Qualixar OS — 2604.06392](https://arxiv.org/abs/2604.06392)

**新闻与分析：**
- [InfoQ — OpenAI Harness Engineering](https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/)
- [InfoQ — Anthropic Three-Agent Harness](https://www.infoq.com/news/2026/04/anthropic-three-agent-harness-ai/)
- [The New Stack — Harness is the Product](https://thenewstack.io/ai-agent-harness-pricing-split/)

**GitHub 项目：**
- [EleutherAI/lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness)
- [affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code)
- [obra/superpowers](https://github.com/obra/superpowers)
- [HKUDS/OpenHarness](https://github.com/HKUDS/OpenHarness)
- [NousResearch/hermes-agent-self-evolution](https://github.com/NousResearch/hermes-agent-self-evolution)
- [coleam00/Archon](https://github.com/coleam00/Archon)
- [aiming-lab/AutoHarness](https://github.com/aiming-lab/AutoHarness)
- [ultraworkers/claw-code](https://github.com/ultraworkers/claw-code)

**社区讨论：**
- [HN — Harnesses Explained (47885131)](https://news.ycombinator.com/item?id=47885131)
- [HN — Importance of Agent Harness (46503379)](https://news.ycombinator.com/item?id=46503379)
- [Claude Code Source Leak 分析](https://alex000kim.com/posts/2026-03-31-claude-code-source-leak/)
