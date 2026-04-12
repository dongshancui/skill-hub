# 企业 Agent Harness Engineering 实践指南

> 基于 [awesome-agent-harness](https://github.com/Picrew/awesome-agent-harness)（141 个资源，最后验证 2026-04-08）深度研读，针对**存量多代码仓项目**的需求→开发→测试→上线全流程场景整理。

---

## 核心认知：Harness 不是框架

> "Frameworks provide abstractions; harnesses provide deterministic execution environments, state management, and recovery mechanisms."

**框架提供抽象，Harness 提供基础设施。** 它解决的核心问题是：当 AI Agent 真正进入企业生产流程后，如何保证它**可靠、可审计、可恢复、可约束**。

对于存量多代码仓 + 全流程场景，核心挑战是：

- 多仓之间上下文割裂，Agent 不知道全貌
- 需求→开发→测试→上线，每个阶段需要不同的 Agent 能力
- 存量代码没有 Agent 友好的结构化描述
- 企业数据安全与权限边界不能妥协

---

## 九层 Harness 架构

### Layer 1：仓库契约层（AGENTS.md）

**最小侵入、最高收益的起点。** 每个代码仓根目录放一个 `AGENTS.md`，作为 Agent 的行为约束合同。

```markdown
# AGENTS.md（每仓必备模板）

## 仓库职责
此服务负责 XX，对外暴露 REST API，数据库使用 PostgreSQL。

## 技术栈约束
- Java 17 + Spring Boot 3.x
- 禁止直接操作 DB，走 Repository 层
- 日志使用 SLF4J，禁止 System.out

## 开发规范
- 新增接口必须有对应 Service 单测，覆盖率 > 80%
- SQL 变更需在 /db/migration/ 新增 Flyway 脚本

## 测试命令
- 单元测试: mvn test
- 集成测试: mvn verify -P integration

## 上线前检查
- 必须通过 CI: ./scripts/pre-check.sh
```

**参考资源**：GitHub Spec Kit、AGENTS.md 开放格式规范

---

### Layer 2：编排层（Orchestration）

面向全流程，使用图状工作流（LangGraph / deepagents）编排各阶段 Agent：

```
需求 Issue
    │
    ▼
[需求理解 Agent] ──── 读取相关仓库 AGENTS.md + 历史代码 + 领域术语
    │
    ▼
[方案设计 Agent] ──── 输出技术方案文档 + 影响范围分析（多仓）
    │
    ▼
[编码 Agent]     ──── 在沙箱中执行，约束在指定仓库范围内
    │
    ▼
[测试 Agent]     ──── 运行已有测试 + 生成新测试
    │
    ▼
[Review Agent]   ──── Diff 审查 + 安全扫描 + 规范检查
    │
    ▼
[上线 Agent]     ──── 生成 PR + 触发 CI/CD（不直接 merge）
```

**关键原则**：每个节点都是有约束的，不是开放式 prompt，而是带 schema validation 的 tool call。

**参考资源**：LangGraph（图状状态机）、deepagents（LangChain 编排层）、DeerFlow（字节，59K⭐）

---

### Layer 3：上下文工程层（Context Engineering）

存量多代码仓最大的痛点。需要主动管理 Agent 的"工作记忆"。

**a. 仓库记忆文件体系**

```
/memory/
  ├── repo-user-service.md      # 仓库级记忆：架构、坑点、关键逻辑
  ├── repo-order-service.md
  ├── cross-repo-deps.md        # 跨仓依赖关系图
  └── domain-glossary.md        # 业务术语表（防止 Agent 误解业务词汇）
```

每次 Agent 启动时，根据任务涉及的仓库，自动注入对应记忆文件。

**b. Context 预算管理（防止 token 爆炸导致 Agent 乱跑）**

```
总 Context Budget: 200K tokens
  ├── 系统 Harness 指令:      5K  （固定）
  ├── AGENTS.md（相关仓库）:  10K （按需加载）
  ├── 代码上下文（检索）:     80K （按需）
  ├── 任务记忆:               20K （会话持久）
  └── 当前操作缓冲:           剩余  （动态）
```

**参考资源**：everything-claude-code、claude-mem（插件化记忆层）、Trellis（多平台记忆注入）

---

### Layer 4：执行沙箱层（Sandboxing）

**Agent 绝对不能直接操作生产仓库或生产数据。** 分层隔离方案：

```
开发阶段: 云沙箱（E2B / Daytona）
  ├── 每次任务创建独立 Git worktree（隔离分支）
  ├── 限制网络访问（只能访问内部 registry）
  └── 文件操作白名单（只能改指定仓库路径）

测试阶段: Docker 容器隔离
  ├── 镜像构建 + 单测执行
  └── 集成测试（只读生产数据）

上线阶段: 人工审批门禁
  └── Agent 只生成 PR，merge 必须人工确认
```

**参考资源**：Daytona（71K⭐，弹性沙箱基础设施）、E2B（云端安全执行）、OpenSandbox（阿里，企业级）

---

### Layer 5：协议标准层（MCP）

将企业内部系统包装成 MCP Server，让所有 Agent 以统一协议调用内部工具：

```
内部 MCP Servers（一次建设，全部 Agent 复用）:
  ├── mcp-jira          # 需求管理：读取 Issue、更新状态、关联代码
  ├── mcp-gitlab        # 代码仓库操作：PR、分支、文件读写
  ├── mcp-nexus         # 内部包管理：依赖查询
  ├── mcp-jenkins       # CI/CD 触发与状态查询
  ├── mcp-sonar         # 代码质量检测结果
  └── mcp-nacos         # 配置中心读取
```

这是企业 Harness 最核心的基础设施投资，建议优先建设。

**参考资源**：MCP 官方规范（83K⭐）、MCP Servers 生态集合

---

### Layer 6：评估层（Evaluation Harness）

**不能靠"感觉"判断 Agent 好不好用——要有数据。**

```yaml
# eval-config.yaml（Promptfoo 风格示例）
scenarios:
  - name: "需求转代码"
    inputs:
      - jira_issue: "USER-123"
      - target_repo: "user-service"
    assertions:
      - type: code_compiles
      - type: tests_pass
      - type: no_security_violation
      - type: follows_coding_standard
        threshold: 0.9
      - type: human_review_score
        min: 3.5
```

**需要建立的关键指标**：

| 指标 | 含义 | 目标方向 |
|---|---|---|
| 任务成功率 | 首次通过 CI 的比例 | 越高越好 |
| 平均 token 消耗/任务 | 成本控制 | 越低越好 |
| 人工干预率 | Agent 自主完成比例 | 逐步降低，但不追求 0 |
| 上线后缺陷率 | Agent 生成代码质量 | 不高于人工基线 |

**参考资源**：Promptfoo（配置驱动测试）、DeepEval、SWE-bench（软件工程基准）、Terminal-Bench

---

### Layer 7：可观测性层（Observability）

每个 Agent 执行链路必须可追踪，方便排查问题和持续优化：

```
Trace ID
  ├── Span: 需求理解      [输入 token: 8K] [耗时: 12s] [状态: OK]
  ├── Span: 方案生成      [输入 token: 15K] [耗时: 28s] [状态: OK]
  ├── Span: 编码执行      [工具调用: 47次] [耗时: 3m12s] [状态: OK]
  ├── Span: 测试运行      [测试数: 124] [失败: 2] [状态: WARN]
  └── Span: PR 生成       [文件变更: 8] [状态: OK]
```

推荐 **Langfuse**（开源可自建）：
- Prompt 版本管理
- 完整执行 trace 记录
- 人工评分反哺优化
- 成本统计

**参考资源**：Langfuse（24K⭐）、Opik（端到端调试）、TensorZero（统一 LLMOps 栈）

---

### Layer 8：Guardrails 层（安全与治理）

企业环境的硬性红线与软性警告：

```python
# 硬性 Guardrails（直接阻断）
FORBIDDEN_OPERATIONS = [
    "直接操作生产数据库",
    "读取 .env / secrets / credentials 文件",
    "修改 CI/CD Pipeline 配置（需额外审批流程）",
    "跨团队仓库的写操作（无授权时）",
    "删除任何文件（需人工确认）",
]

# 软性 Guardrails（警告 + 人工确认）
WARN_OPERATIONS = [
    "单次修改超过 500 行代码",
    "新增外部第三方依赖",
    "修改数据库 Schema",
    "修改公共基础库",
]
```

---

### Layer 9：参考实现层

建议在企业内部建立一个 **内部 Harness Reference 仓库**，收录：
- 各业务域的 AGENTS.md 标准模板
- 典型任务的 Agent workflow 配置（可复用）
- 成功案例的 trace 记录（脱敏后）
- 失败案例的 post-mortem 与改进方案

**外部参考实现**：Claude Code（111K⭐）、OpenHands、OpenCode（139K⭐）、aider

---

## 落地路线图

### Phase 1：播种期（第 1-2 个月）

**目标：让 Agent 能可靠地跑起来**

- [ ] 选 2-3 个代表性仓库做试点（优先选技术债少、测试覆盖好的）
- [ ] 为每个试点仓库编写 AGENTS.md
- [ ] 部署 Langfuse 可观测性平台
- [ ] 实现最简单的单仓任务（修 Bug 级别，不跨仓）
- [ ] 建立人工评审 SOP：每个 PR 必须人工 review + 打分

### Phase 2：扎根期（第 3-4 个月）

**目标：多仓协作 + 全流程打通**

- [ ] 建设内部核心 MCP Server（jira + gitlab 优先）
- [ ] 实现跨仓上下文记忆系统（memory/ 目录体系）
- [ ] 打通 Jira → 代码 → 测试自动化链路
- [ ] 引入 Promptfoo/DeepEval，建立质量评估基线
- [ ] 沙箱隔离落地（Daytona 或 E2B）

### Phase 3：生长期（第 5-6 个月）

**目标：规模化 + 自我优化**

- [ ] 扩展 AGENTS.md 覆盖全部代码仓
- [ ] 基于 Eval 数据持续优化 Agent workflow
- [ ] 建立团队级 Harness 运营机制（谁负责维护、如何迭代）
- [ ] 常规任务（格式修复、依赖升级、标准 CRUD）实现低干预自动化

---

## 常见陷阱

| 陷阱 | 根本原因 | 对策 |
|---|---|---|
| Agent 改了不该改的文件 | 没有沙箱 + 文件操作白名单 | Layer 4 先于其他层落地 |
| Context 超限，Agent 输出乱 | 没有 token 预算管理 | 明确每层的 context 配额上限 |
| 出了问题不知道哪步错了 | 没有可观测性 | Langfuse 第一天就部署 |
| 多仓风格不一致，Agent 行为飘忽 | AGENTS.md 各自为政 | 建立公司级标准模板 |
| 开发团队不信任 Agent 输出 | 没有评估数据支撑信任建立 | 用 Eval 数据和历史 trace 说话 |
| 需求理解偏差，写出来的不是要的 | Agent 缺乏业务上下文 | domain-glossary.md + 记忆注入 |
| 跳过基础设施直接做"全流程" | 急于求成 | 严格按路线图分阶段，基础设施优先 |

---

## 优先级资源清单

按照本场景的落地优先级排序：

| 优先级 | 资源 | 用途 | Stars |
|---|---|---|---|
| P0 | AGENTS.md 规范 | 仓库契约，最先动手 | — |
| P0 | Langfuse | 可观测性，第一天部署 | 24K⭐ |
| P1 | LangGraph | 全流程编排 | — |
| P1 | Daytona | 沙箱执行隔离 | 71K⭐ |
| P1 | MCP Servers | 内部工具标准化 | 83K⭐ |
| P2 | Promptfoo | Eval 评估体系 | — |
| P2 | claude-mem | 仓库记忆注入 | — |
| P3 | Claude Code | 编码 Agent 参考实现 | 111K⭐ |
| P3 | OpenCode | 终端 Agent 参考实现 | 139K⭐ |

---

## 一句话原则

> 先建基础设施（沙箱 + 可观测性 + AGENTS.md），再做编排，再求规模。  
> 不要跳过基础设施直接做"全流程自动化"——那是在沙地上盖楼。
