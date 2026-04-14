# Harness Engineering 资源汇总

> 最后更新：2026-04-14

---

## GitHub 项目

### 核心框架

- [aiming-lab/AutoHarness](https://github.com/aiming-lab/AutoHarness) — 自动化 Harness Engineering 框架，含三层 pipeline 模式、6 步治理流水线、YAML 宪法、多 agent profile
- [HKUDS/OpenHarness](https://github.com/HKUDS/OpenHarness) — 开源 Agent Harness，内置个人 agent（Ohmo），支持 MCP HTTP transport、多 provider 认证
- [langchain-ai/deepagents](https://github.com/langchain-ai/deepagents) — LangChain + LangGraph 构建的 agent harness，支持规划工具、文件系统后端、子 agent 派生
- [deepklarity/harness-kit](https://github.com/deepklarity/harness-kit) — AI agent 工程化模式套件，不只是编排，更关注工程纪律
- [BulloRosso/etienne](https://github.com/BulloRosso/etienne) — 面向企业环境的 Coding Agent Harness，合规优先
- [adrielp/ai-engineering-harness](https://github.com/adrielp/ai-engineering-harness) — AI agent harness 上下文工程实践
- [revfactory/harness](https://github.com/revfactory/harness) — 元技能框架，设计领域专属 agent 团队与技能

### 数据工程专项

- [AltimateAI/altimate-code](https://github.com/AltimateAI/altimate-code) — 面向 dbt/SQL/云数仓的开源 agentic 数据工程 harness，100+ 工具，10 种仓库

### 评测 / Benchmark

- [harbor-framework/harbor](https://github.com/harbor-framework/harbor) — 通用 agent 评测框架，支持 RL 环境、云容器部署，兼容 Claude Code、Codex CLI、OpenHands 等
- [harbor-framework/terminal-bench](https://github.com/harbor-framework/terminal-bench) — Terminal 任务 LLM benchmark（ICLR 2026 收录）
- [harbor-framework/terminal-bench-2](https://github.com/harbor-framework/terminal-bench-2) — Terminal-Bench 2.0，更严格的验证与任务集
- [harbor-framework/terminal-bench-science](https://github.com/harbor-framework/terminal-bench-science) — 面向复杂科学工作流的 terminal agent 评测
- [badlogic/pi-terminal-bench](https://github.com/badlogic/pi-terminal-bench) — pi coding agent 的 Terminal-Bench Harbor adapter
- [openai/codex discussions #12219](https://github.com/openai/codex/discussions/12219) — Codex harness 在 Terminal-Bench 2.0 中的使用讨论

### Awesome 列表

- [ai-boost/awesome-harness-engineering](https://github.com/ai-boost/awesome-harness-engineering) — Harness Engineering 资源聚合
- [walkinglabs/awesome-harness-engineering](https://github.com/walkinglabs/awesome-harness-engineering) — Harness Engineering 工具与指南合集
- [tmgthb/Autonomous-Agents](https://github.com/tmgthb/Autonomous-Agents) — 自主 agent LLM 论文持续更新列表

---

## 原始博文 / 概念起源

- [Mitchell Hashimoto — My AI Adoption Journey](https://mitchellh.com/writing/my-ai-adoption-journey) — 2026-02-05，"harness engineering"概念首次命名，来自每次 agent 出错后固化修复到环境中的习惯
- [Martin Fowler — Harness engineering for coding agent users](https://martinfowler.com/articles/harness-engineering.html) — 2026-04-02，系统化阐述 Harness 构成：guides、sensors、计算/推理组件、harness 模板
- [Martin Fowler — Harness Engineering first thoughts (memo)](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering-memo.html) — 2026-02-17，最初备忘录
- [Martin Fowler — Humans and Agents in Software Engineering Loops](https://martinfowler.com/articles/exploring-gen-ai/humans-and-agents.html) — 人类与 agent 协作循环的上下文

---

## 权威机构文章

- [OpenAI — Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/) — OpenAI 官方，Codex 五个月生成百万行代码的 harness 实践
- [OpenAI Introduces Harness Engineering: Codex Agents Power Large-Scale Software Development — InfoQ](https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/) — InfoQ 报道，2026-02

---

## Anthropic 相关

- [Anthropic's New Harness Engineering Playbook — Medium](https://medium.com/ai-software-engineer/anthropics-new-harness-engineering-playbook-shows-why-your-ai-agents-keep-failing-174a5575ff92) — Anthropic harness 设计解析
- [Anthropic's Harness Engineering: Two Agents, One Feature List, Zero Context Overflow — Medium](https://medium.com/@richardhightower/anthropics-harness-engineering-two-agents-one-feature-list-zero-context-overflow-7c26eb02c807) — 多 agent 上下文溢出解法

---

## 概念解析 / 综合指南

- [Agentic Harness Engineering: LLMs as the New OS — Decoding AI](https://www.decodingai.com/p/agentic-harness-engineering)
- [Beyond Prompts and Context: Harness Engineering for AI Agents — MadPlay](https://madplay.github.io/en/post/harness-engineering)
- [The Third Evolution: Why Harness Engineering Replaced Prompting in 2026 — Epsilla](https://www.epsilla.com/blogs/harness-engineering-evolution-prompt-context-autonomous-agents)
- [Harness Engineering: The Missing Layer Behind AI Agents — Louis Bouchard](https://www.louisbouchard.ai/harness-engineering/)
- [Harness Engineering Is Not Context Engineering — mtrajan substack](https://mtrajan.substack.com/p/harness-engineering-is-not-context)
- [Harness engineering: why context beats intelligence — bitsofchris](https://bitsofchris.com/p/harness-engineering-why-context-beats)
- [What is Agent Context Engineering? And How is it Different? — Promptless](https://promptless.ai/blog/technical/agent-context-engineering/)
- [2025 Was Agents. 2026 Is Agent Harnesses. — Aakash Gupta / Medium](https://aakashgupta.medium.com/2025-was-agents-2026-is-agent-harnesses-heres-why-that-changes-everything-073e9877655e)
- [The Rise of AI Harness Engineering — Cobus Greyling / Medium](https://cobusgreyling.medium.com/the-rise-of-ai-harness-engineering-5f5220de393e)
- [The Rise of AI Harness Engineering — Cobus Greyling / Substack](https://cobusgreyling.substack.com/p/the-rise-of-ai-harness-engineering)
- [What we miss when we talk about "AI Harnesses" — Future of Being Human](https://www.futureofbeinghuman.com/p/what-we-miss-when-we-talk-about-ai-harnesses)
- [Decode the Buzzword: Why Harness Engineering Matters Now — Next Signal Prediction](https://nextsignalprediction.substack.com/p/decode-the-buzzword-why-harness-engineering)
- [Mass Programming Resistance — Harness Engineering](https://mpr.crossjam.net/wp/mpr/2026/02/harness-engineering/)
- [Skill Issue: Harness Engineering for Coding Agents — HumanLayer](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents)

---

## 入门 / 教程类

- [What Is Harness Engineering for AI Agents? — Milvus Blog](https://milvus.io/blog/harness-engineering-ai-agents.md)
- [Harness Engineering: Uncovering What It Is — Data Science Dojo](https://datasciencedojo.com/blog/harness-engineering/)
- [What is Harness Engineering? A Complete Introduction (2026) — Harness Engineering Academy](https://harnessengineering.academy/blog/what-is-harness-engineering-introduction-2026/)
- [The Complete Guide to Agent Harness — harness-engineering.ai](https://harness-engineering.ai/blog/agent-harness-complete-guide/)
- [Complete Guide to Harness Engineering — QubitTool](https://qubittool.com/blog/harness-engineering-complete-guide)
- [Agent Harness Engineering Guide [2026] — QubitTool](https://qubittool.com/blog/agent-harness-evaluation-guide)
- [What Is Harness Engineering? Guide to Reliable AI Agents — agent-engineering.dev](https://www.agent-engineering.dev/article/harness-engineering-in-2026-the-discipline-that-makes-ai-agents-production-ready)
- [What Is Harness Engineering? Complete Guide (2026) — NxCode](https://www.nxcode.io/resources/news/what-is-harness-engineering-complete-guide-2026)
- [Harness Engineering: The Complete Guide (2026) — NxCode](https://www.nxcode.io/resources/news/harness-engineering-complete-guide-ai-agent-codex-2026)
- [What Is Harness Engineering: A New Concept — SmartScope](https://smartscope.blog/en/blog/harness-engineering-overview/)
- [Agent Harness: What Actually Determines Whether AI Delivers or Disappoints — WenHao Yu](https://yu-wenhao.com/en/blog/ai-harness/)
- [Harness Engineering: Why Your AI Agents Keep Failing in Production — Shulex VOC](https://blog.voc.ai/harness-engineering-agent-architecture/)
- [What Is Harness Engineering? The New Discipline — AgentBoard](https://agentboard.cc/blog/what-is-harness-engineering)
- [Harness Engineering: Your Job Isn't Writing Code Anymore — Vibe Sparking AI](https://www.vibesparking.com/en/blog/ai/context-engineering/2026-03-06-harness-engineering-agents-first-world/)

---

## 学术论文

- [Building AI Coding Agents for the Terminal: Scaffolding, Harness, Context Engineering — arXiv](https://arxiv.org/html/2603.05344v1)
- [Terminal-Bench: Benchmarking (ICLR 2026) — OpenReview PDF](https://openreview.net/pdf/417ac3236de7dbf3fc3414c51754dd239271663e.pdf)
- [Harness Engineering for Language (preprint) — Preprints.org](https://www.preprints.org/frontend/manuscript/567757f184a1af99de64c01b54a2d366/download_pub)

---

## Benchmark / 评测站点

- [Terminal-Bench 官网](https://www.tbench.ai/)
- [Terminal-Bench 2.0 & Harbor 发布公告](https://www.tbench.ai/news/announcement-2-0)
