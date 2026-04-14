# Harness Engineering 深度阅读总结

> 本文为 [harness-engineering-resources.md](./harness-engineering-resources.md) 所有来源的阅读摘要。每个结论均标注原始来源，方便追溯。
> 整理时间：2026-04-14

---

## 一、概念起源：这个词从哪来的

"Harness Engineering"这个词由 **Mitchell Hashimoto**（Ghostty、HashiCorp 创始人）在 2026 年 2 月 5 日首次命名。

> **原文引用（[Mitchell Hashimoto — My AI Adoption Journey](https://mitchellh.com/writing/my-ai-adoption-journey)）：**
> "anytime you find an agent makes a mistake, you take the time to engineer a solution such that the agent never makes that mistake again."

他的发现来自一个简单观察：

> "agents are much more efficient when they produce the right result the first time, or at worst produce a result that requires minimal touch-ups."

他在自己的 Ghostty 项目里实践这一理念——每次 agent 出错，就把修复方案固化进 `AGENTS.md`。他说那个文件里每一行都是一个真实的 agent 失误，"almost completely resolved them all"。

**两种落地方式：**
1. 写进 `AGENTS.md` / `CLAUDE.md`，让 agent 下次读到就知道不该这么做
2. 写成工具脚本，让 agent 能自动验证自己的工作

---

## 二、三个阶段的演进：从 Prompt 到 Harness

来源：[The Third Evolution: Why Harness Engineering Replaced Prompting in 2026 — Epsilla](https://www.epsilla.com/blogs/harness-engineering-evolution-prompt-context-autonomous-agents)

| 阶段 | 时间 | 核心问题 | 做法 |
|------|------|---------|------|
| **Prompt Engineering** | 2022–2024 | 怎么问？ | 精心设计单次指令，few-shot、chain-of-thought |
| **Context Engineering** | 2025 | 给模型看什么？ | 动态构建上下文窗口，RAG、历史、工具定义 |
| **Harness Engineering** | 2026 | 整个系统怎么运转？ | 约束、反馈循环、工具链、生命周期管理 |

这不是替代关系，而是嵌套关系——harness 包含 context，context 包含 prompt。

---

## 三、Harness 到底是什么

### 3.1 最简公式

来源：[Martin Fowler — Harness engineering for coding agent users](https://martinfowler.com/articles/harness-engineering.html)、[Decoding AI — Agentic Harness Engineering](https://www.decodingai.com/p/agentic-harness-engineering)

> **Agent = Model + Harness**

模型提供智能，Harness 提供"手、眼睛、记忆和安全边界"。

> "An agent equals a model plus a harness. The harness is every piece of code, configuration, and execution logic that is not the model itself."（Decoding AI）

### 3.2 Harness 的构成层次（Martin Fowler 框架）

来源：[Martin Fowler — Harness engineering for coding agent users](https://martinfowler.com/articles/harness-engineering.html)

Birgitta Böckeler（Thoughtworks Distinguished Engineer）系统化了 harness 的结构：

**两种控制机制：**
- **Guides（前馈控制）**：在 agent 行动之前介入，引导 agent 第一次就做对
- **Sensors（反馈控制）**：在 agent 行动之后观察，触发自我纠正

**两种执行类型：**
- **Computational（确定性）**：毫秒级，如测试、linter、类型检查
- **Inferential（推理性）**：通过 AI 做语义分析，如 code review agent、LLM as judge

**三类 Harness：**
1. **Maintainability Harness**：管代码内部质量，用现有工具链
2. **Architecture Fitness Harness**：定义和检查架构特性，通过适应性函数执行
3. **Behaviour Harness**：管功能正确性——目前最不成熟，还需更多研究

> "A good harness should not necessarily aim to fully eliminate human input, but to direct it to where our input is most important."

### 3.3 Harness 的六大核心组件

来源：[Cobus Greyling — The Rise of AI Harness Engineering](https://cobusgreyling.medium.com/the-rise-of-ai-harness-engineering-5f5220de393e)

> "A harness is not the agent. It's the software system that governs how the agent operates."

六个组件：
1. **Tool Integration Layer** — 通过协议将模型接入外部系统
2. **Memory and State Management** — 跨上下文窗口的多层记忆持久化
3. **Context Engineering** — 每次调用时动态选择相关信息
4. **Planning and Decomposition** — 将任务结构化而非单次整体尝试
5. **Verification and Guardrails** — 验证、安全过滤、自我纠正循环
6. **Modularity and Extensibility** — 可插拔、可独立替换的组件

---

## 四、Harness Engineering ≠ Context Engineering

来源：[Harness Engineering Is Not Context Engineering — mtrajan](https://mtrajan.substack.com/p/harness-engineering-is-not-context)

这是当前最容易混淆的两个概念，区别如下：

> "Context engineering asks what should the agent see. Harness engineering is about what should the system prevent, measure, and correct."

> "Context helps the model think. Harness is what keeps the system from drifting."

OpenAI 内部的实验提供了最直接的证据：团队早期每周花约"a day a week of cleanup"处理 AI 产出的垃圾（"AI slop"），直到他们把标准**直接编码进仓库基础设施**才解决——这就是从 context engineering 到 harness engineering 的跨越。

### 类比

来源：[Beyond Prompts and Context: Harness Engineering for AI Agents — MadPlay](https://madplay.github.io/en/post/harness-engineering)

> 如果 prompt 是命令"右转"，context 是地图和路标，那么 harness 就是缰绳、马鞍、围栏和道路本身。

---

## 五、关键数据与证明

来源：[MadPlay](https://madplay.github.io/en/post/harness-engineering)、[Aakash Gupta — 2025 Was Agents. 2026 Is Agent Harnesses.](https://aakashgupta.medium.com/2025-was-agents-2026-is-agent-harnesses-heres-why-that-changes-everything-073e9877655e)、[InfoQ OpenAI 报道](https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/)

**最震撼的三个实验数据：**

| 实验 | 做了什么 | 结果 |
|------|---------|------|
| **OpenAI Codex 内部实验** | 5 个月，只改 harness，不改模型 | ~100 万行生产代码，"roughly 10x faster" |
| **Hashline 实验** | 只改 edit format（harness 的一部分），不改模型权重 | 准确率从 6.7% → **68.3%** |
| **LangChain** | 只改 harness 组件，不改模型 | benchmark 从 52.8% → **66.5%**，进入 Top 5 |

**更多证据（来源：Aakash Gupta）：**
- **Manus**：用完全相同的模型，6 个月内重写了 5 次 harness，每次都显著提升可靠性
- **Vercel**：移除 80% 的 agent 工具后结果反而更好——"harness improvement through subtraction"

> "The model is commodity. The harness is moat."（Aakash Gupta）

---

## 六、LLM 即新操作系统

来源：[Decoding AI — Agentic Harness Engineering: LLMs as the New OS](https://www.decodingai.com/p/agentic-harness-engineering)

这篇文章提出了一个重要类比：LLM 正在成为新的操作系统，harness 是运行在其上的基础设施。

**ReAct 规划循环（Harness 的运行引擎）：**
观察状态 → 推理下一步 → 通过工具执行 → 观察结果 → 循环

**三层记忆架构：**
- **Filesystem**：长期持久状态（主干）
- **RAM**：快速易失工作记忆
- **Context window**：模型实际能看到的内容

> "Feedback loops improve quality by two to three times"——agent 需要验证自己工作的机制。

> "The model was never the problem. The system and infrastructure around it" 决定了 agent 是否在生产中真正可用。

---

## 七、OpenAI 实践（via InfoQ + Anthropic Playbook）

来源：[InfoQ — OpenAI Introduces Harness Engineering](https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/)、[Anthropic's New Harness Engineering Playbook — Medium](https://medium.com/ai-software-engineer/anthropics-new-harness-engineering-playbook-shows-why-your-ai-agents-keep-failing-174a5575ff92)

**OpenAI 的 Harness 三层架构：**
1. **Context Engineering**：增强知识库 + 动态上下文（可观测性、浏览器导航）
2. **Architectural Constraints**：确定性 linter + 结构测试 + LLM agent 并行
3. **Garbage Collection**：定期运行的 agent，对抗熵增和文档漂移

> "no manually typed code at all"——零手写代码的约束贯穿整个实验

Ryan Lopopolo（OpenAI）：整个系统构建是为了"teams can focus on research and product development rather than infrastructure orchestration."

**Anthropic 的发现：**

> "Long-running autonomous tasks are the most consistent fail point of even the most polished AI agents."

Anthropic 发布了两篇工程论文：第一篇（2025-11）记录了基础 harness 设计原则，第二篇（2026-03）提供了长时间运行应用的详细实施指南。

---

## 八、Coding Agent 的 Harness 实操

来源：[HumanLayer — Skill Issue: Harness Engineering for Coding Agents](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents)

这是最接地气的实操指南，结论是：

> "it's not a model problem. It's a **configuration problem**."

**六个配置杠杆：**

**1. CLAUDE.md / AGENTS.md**
> "Less (instructions) is more." 作者自己的文件不超过 60 行。

ETH Zurich 研究数据：
- 自动生成的文件：**损害性能 20%+**
- 精心手写的文件：**提升约 4%**

**2. MCP Servers（工具扩展）**
工具描述注入 system prompt，是 prompt injection 的危险入口。**工具太多会让 agent 进入"dumb zone"**——上下文窗口被工具定义撑爆。

**3. Skills（渐进式加载）**
按需加载，避免一次性耗尽 instruction budget。技能目录可以包含多个 markdown 文件，灵活组合。

**4. Sub-Agents（上下文隔离）**
有效的子 agent 不应按角色分（"前端工程师"、"后端工程师"），而应按**离散任务**划分，在独立上下文窗口里执行，形成"context firewall"。

研究证实：即使在简单任务上，上下文越长模型性能越差。

**5. Hooks（控制流）**
在生命周期事件自动触发——类型检查、格式化、审批、集成。**成功无声，只有失败才向 agent 报告。**

**6. Back-Pressure（验证机制）**
让 agent 能自己验证工作。关键洞察：
> "success is silent, and only failures produce verbose output."

**总体策略：bias toward shipping**——先简单上线，只在 agent 真正失败时才加配置，优化迭代速度而非追求第一次完美。

---

## 九、OpenAI 与 Anthropic 的多 Agent 实践

来源：[Anthropic's Harness Engineering: Two Agents — Medium](https://medium.com/@richardhightower/anthropics-harness-engineering-two-agents-one-feature-list-zero-context-overflow-7c26eb02c807)

**Anthropic 解决上下文溢出的方法**：用两个 agent 分担一个 feature list——不是让一个 agent 做完所有事，而是把任务切分成独立的上下文单元，每个 agent 持有足够小的上下文完成自己的部分。

这本质上是用 **harness 的任务分解能力** 绕过模型的 context window 限制。

---

## 十、论文层面的定义（arXiv 2603.05344）

来源：[Building AI Coding Agents for the Terminal — arXiv](https://arxiv.org/html/2603.05344v1)

这篇论文对三个概念做了最严格的学术区分：

| 概念 | 阶段 | 定义 |
|------|------|------|
| **Scaffolding** | 执行前 | 预构建：编译 system prompt、注册工具、初始化 subagent |
| **Harness** | 运行时 | 编排：工具分发、上下文管理、安全执行、会话持久化 |
| **Context Engineering** | 持续 | token 效率：系统提醒、模块化 prompt、自适应压缩、跨会话记忆 |

**创新点：**
- **复合 AI 系统**：五种专用模型角色（正常执行/推理/自我批评/视觉/fallback），每个工作流独立选模型，实现成本-延迟-能力的精细调优
- **五层纵深防御安全体系**：prompt 护栏 → schema 限制 → 运行时审批 → 工具级验证 → 生命周期钩子，无单点故障
- **双模式**：Plan Mode（只读工具）和 Normal Mode（完整工具）在 schema 层面强制分离

---

## 十一、主要开源框架对比

### AutoHarness（[aiming-lab/AutoHarness](https://github.com/aiming-lab/AutoHarness)）

定位：轻量级 agent 治理框架，让每个 agent 都能"有 aha moment"（从原型到可靠部署的跨越）。

核心是 **6 步治理 pipeline**，每次工具调用都流经：
`Parse & Validate → Risk Classify → Permission Check → Execute → Output Sanitize → Audit Log`

三种模式（6/8/14 步）对应不同场景。亮点：
- 只需 2 行代码包装现有 OpenAI SDK 客户端
- YAML 宪法（声明式治理规则）
- 每次调用的精确成本追踪
- JSONL 完整审计日志

### OpenHarness（[HKUDS/OpenHarness](https://github.com/HKUDS/OpenHarness)）

定位：开源 Python agent 基础设施 + 内置个人 agent（ohmo）。

五大支柱：Agent Loop Engine、43+ 工具生态、知识系统（按需加载 skill）、安全框架（多级权限）、多 agent 协调。

特点：ohmo 可以在 Feishu/Slack/Telegram/Discord 上运行，自动 fork 分支、写代码、跑测试、开 PR，不需要额外 API key。MCP 是一等公民，支持 HTTP transport 和自动重连。

### langchain-ai/deepagents（[langchain-ai/deepagents](https://github.com/langchain-ai/deepagents)）

LangChain + LangGraph 构建，内置规划工具、文件系统后端、子 agent 派生能力，适合需要基于成熟框架快速搭建的场景。

---

## 十二、评测生态：Terminal-Bench + Harbor

来源：[Terminal-Bench 2.0 & Harbor 发布公告](https://www.tbench.ai/news/announcement-2-0)、[harbor-framework/harbor](https://github.com/harbor-framework/harbor)

**Terminal-Bench**：已被各前沿 AI 实验室采用的 agent 评测标准（ICLR 2026 收录），测试 LLM 在终端执行复杂任务的能力。

**Terminal-Bench 2.0** 解决了 1.0 版任务质量参差的问题："Substantial manual and LM-assisted verification went into the creation of each task"，实验室评价为"some of the highest quality environments they have seen"。

**Harbor 框架**：与 2.0 同期发布，解决三个核心问题：
1. **Scale**：水平扩展到数千个云部署容器
2. **Improvement**：支持 SFT、RL、prompt 优化的 agent 改进接口
3. **Generalization**：兼容任何可以装进容器的 agent（Claude Code、Codex CLI、OpenHands 等均已支持）

Terminal-Bench 1.0 发布后积累了 1000+ Discord 成员和 100+ GitHub 贡献者，成为事实标准。

---

## 十三、行业影响与趋势

来源：[Martin Fowler memo](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering-memo.html)、[Cobus Greyling](https://cobusgreyling.medium.com/the-rise-of-ai-harness-engineering-5f5220de393e)、[Aakash Gupta](https://aakashgupta.medium.com/2025-was-agents-2026-is-agent-harnesses-heres-why-that-changes-everything-073e9877655e)

**Martin Fowler 的三个假设：**
1. **Tech Stack 收敛**：AI 采用可能减少技术栈多样性，有强 harness 支持的栈更受青睐
2. **架构刚性**：可维护性越来越依赖标准化模式和强制边界
3. **改造成本**：给遗留代码库加 harness 代价高昂，新应用从一开始就有 harness 反而有优势

**框架层面的演变（Cobus Greyling）：**
> 传统框架约 80% 的职责（agent 定义、路由、编排）已被能力强的模型原生吸收；剩余 20%——持久化、成本控制、可观测性、错误恢复——才是 harness 的真正领域。

**竞争护城河（Aakash Gupta）：**
> 构建生产级 harness 需要"thousands of engineering hours"（至少 6 个月），而微调一个有竞争力的模型只需几周。先动手的团队建立起竞争者难以追赶的结构性优势。

---

## 总结：三句话理解 Harness Engineering

1. **"The model was never the problem."** — Harness Engineering 的出发点：模型够用了，是周围的系统不行
2. **"Every mistake becomes a permanent fix."** — Mitchell Hashimoto 的核心实践：每个错误都化为对环境的改造
3. **"The model is commodity. The harness is moat."** — 当模型能力趋同，harness 设计是真正的差异化优势
