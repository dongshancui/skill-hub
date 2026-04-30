# opencode `.opencode` 目录完整指南

## 目录结构总览

```
.opencode/
├── agents/       # 自定义 AI 助手（每个文件一个 agent）
├── commands/     # 自定义命令（/xxx 命令）
├── modes/        # 自定义模式
├── plugins/      # 插件
├── skills/       # 可复用技能包
├── tools/        # 自定义工具
└── themes/       # 主题
```

全局等效路径：`~/.config/opencode/[上述目录名]/`

目录名支持复数形式（`agents/`），也向下兼容单数形式（`agent/`）。

---

## 配置加载优先级

配置按以下顺序加载，后者覆盖前者（不冲突的字段会合并保留）：

1. 远程配置（`.well-known/opencode`）— 组织级默认值
2. 全局配置（`~/.config/opencode/opencode.json`）
3. 自定义配置（`OPENCODE_CONFIG` 环境变量）
4. 项目配置（项目根目录 `opencode.json`）
5. `.opencode` 目录内容（agents、commands、plugins 等）
6. 内联配置（`OPENCODE_CONFIG_CONTENT` 环境变量）
7. 托管配置（系统目录）
8. macOS MDM 配置（最高优先级）

---

## 1. `agents/` — 自定义 Agent

每个 `.md` 文件就是一个 agent，文件名即 agent ID。

```markdown
<!-- .opencode/agents/review.md -->
---
description: 审查代码质量和最佳实践
mode: subagent
model: anthropic/claude-sonnet-4-5
temperature: 0.1
permission:
  edit: deny
  bash: deny
---
你是代码审查专家，专注于安全性、性能和可维护性。
```

### 可配置字段

| 字段 | 说明 |
|---|---|
| `description` | 必填，agent 用途描述 |
| `mode` | `primary`（主对话）/ `subagent`（被调用）/ `all` |
| `model` | 覆盖默认模型 |
| `temperature` | 0.0–1.0，越低越确定 |
| `steps` | 最大执行步数（原 `maxSteps` 已弃用） |
| `permission` | 各工具权限：`ask` / `allow` / `deny` |
| `color` | 颜色标识（支持 hex 或主题色名） |
| `hidden` | 隐藏于 @ 自动补全 |
| `disable` | 设为 `true` 禁用此 agent |
| `top_p` | 采样多样性控制 |

### 权限可控工具

| 权限键 | 控制的工具 |
|---|---|
| `read` | `read` |
| `edit` | `write`, `edit`, `apply_patch` |
| `glob` | `glob` |
| `grep` | `grep` |
| `bash` | `bash` |
| `task` | `task` |
| `webfetch` | `webfetch` |
| `websearch` | `websearch` |
| `skill` | `skill` |
| `question` | `question` |

bash 可细粒度控制：
```json
{
  "permission": {
    "bash": {
      "git push": "ask",
      "grep *": "allow",
      "*": "ask"
    }
  }
}
```

### 内置 Agent

**主 Agent（Tab 切换）：**
- `build` — 默认，全工具开放
- `plan` — 规划模式，`edit` 和 `bash` 默认为 `ask`

**子 Agent（@ 调用）：**
- `general` — 通用研究，全工具（除 todo）
- `explore` — 快速只读代码探索，不可修改文件

**隐藏系统 Agent（自动运行）：**
- `compaction` — 压缩长上下文
- `title` — 生成会话标题
- `summary` — 创建会话摘要

---

## 2. `commands/` — 自定义命令

文件名 → `/命令名`（如 `test.md` → `/test`）。

```markdown
<!-- .opencode/commands/test.md -->
---
description: 运行完整测试套件
agent: build
model: anthropic/claude-haiku-4-5
subtask: false
---
运行带覆盖率报告的完整测试套件，并展示失败的测试。

当前测试输出：
!`npm test`

基于以上结果，给出改进建议。
```

### 配置选项

| 选项 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `template` | string | 是 | 发送给 LLM 的提示词 |
| `description` | string | 否 | TUI 中显示的描述 |
| `agent` | string | 否 | 指定使用的 agent |
| `subtask` | boolean | 否 | 强制以子任务方式调用 |
| `model` | string | 否 | 覆盖默认模型 |

### 模板特性

- `$ARGUMENTS`、`$1`、`$2`... — 接收命令参数
- `` !`command` `` — 嵌入 shell 命令输出（从项目根目录执行）
- `@filename` — 引用文件内容

---

## 3. `skills/` — 可复用技能

每个技能是一个子目录，内含 `SKILL.md`（文件名必须全大写）：

```
.opencode/skills/git-release/SKILL.md
```

```yaml
---
name: git-release
description: 创建一致的发布版本和 changelog
license: MIT
compatibility: opencode
metadata:
  audience: maintainers
---

## 我做什么
- 从合并的 PR 草拟发布说明
- 提议版本号
- 提供可复制的 `gh release create` 命令

## 何时使用
准备打 tag 发布时使用。
```

### 文件名规范

- `name` 字段必须与所在目录名一致
- 1–64 字符，小写字母数字，单连字符分隔
- 不能以 `-` 开头或结尾，不能有 `--`

### 技能发现路径

OpenCode 按以下路径查找技能（从当前目录向上遍历至 git worktree）：
- `.opencode/skills/<name>/SKILL.md`
- `.claude/skills/<name>/SKILL.md`
- `.agents/skills/<name>/SKILL.md`
- `~/.config/opencode/skills/<name>/SKILL.md`
- `~/.claude/skills/<name>/SKILL.md`

### 权限控制

```json
{
  "permission": {
    "skill": {
      "*": "allow",
      "internal-*": "deny",
      "experimental-*": "ask"
    }
  }
}
```

---

## 4. `plugins/` — 插件

插件扩展 OpenCode 的工具、钩子和集成能力。

```json
{
  "plugin": ["opencode-helicone-session", "@my-org/custom-plugin"]
}
```

也可直接将插件文件放入 `.opencode/plugins/`，启动时自动加载。

---

## 5. `tools/` — 自定义工具

让 LLM 可调用自定义函数：

```
.opencode/tools/        # 项目级
~/.config/opencode/tools/  # 全局级
```

---

## 6. `AGENTS.md` — 项目规则

放在项目根目录，相当于 Claude Code 的 `CLAUDE.md`：

```markdown
# 项目规则

## 构建命令
- 构建：`bun run build`
- 测试：`bun test`
- 代码检查：`bun lint`

## 代码规范
- 使用 TypeScript strict 模式
- 所有注释和文档用中文
- 共享代码放 `packages/core/`

## 目录结构
- `packages/` — 工作区包
- `infra/` — 基础设施定义
```

### 规则加载优先级

1. 项目 `AGENTS.md`（没有则回退到 `CLAUDE.md`）
2. 全局 `~/.config/opencode/AGENTS.md`（没有则回退到 `~/.claude/CLAUDE.md`）

### 通过 `opencode.json` 引用额外规则文件

```json
{
  "instructions": [
    "CONTRIBUTING.md",
    "docs/guidelines.md",
    ".cursor/rules/*.md",
    "https://raw.githubusercontent.com/my-org/shared-rules/main/style.md"
  ]
}
```

支持 glob 模式和远程 URL（5 秒超时）。

---

## 7. `opencode.json` — 核心配置文件

放在项目根目录，支持 JSONC（带注释的 JSON）：

```jsonc
{
  "$schema": "https://opencode.ai/config.json",

  // 模型配置
  "model": "anthropic/claude-sonnet-4-6",
  "small_model": "anthropic/claude-haiku-4-5",

  // 规则文件
  "instructions": ["AGENTS.md"],

  // 工具权限
  "permission": {
    "edit": "ask",
    "bash": "ask"
  },

  // 上下文压缩
  "compaction": {
    "auto": true,
    "prune": true,
    "reserved": 10000
  },

  // 快照（支持撤销）
  "snapshot": true,

  // 自动更新
  "autoupdate": false,

  // 文件监听忽略
  "watcher": {
    "ignore": ["node_modules/**", "dist/**", ".git/**"]
  },

  // MCP 服务器
  "mcp": {},

  // 插件
  "plugin": []
}
```

---

## 与 Claude Code 的兼容性

opencode 原生支持 Claude Code 配置文件作为回退：

| opencode 原生 | Claude Code 回退 |
|---|---|
| `AGENTS.md` | `CLAUDE.md` |
| `~/.config/opencode/AGENTS.md` | `~/.claude/CLAUDE.md` |
| `.opencode/skills/` | `.claude/skills/` |

如需禁用兼容：
```bash
OPENCODE_DISABLE_CLAUDE_CODE=1          # 完全禁用
OPENCODE_DISABLE_CLAUDE_CODE_PROMPT=1   # 仅禁用 ~/.claude/CLAUDE.md
OPENCODE_DISABLE_CLAUDE_CODE_SKILLS=1   # 仅禁用 .claude/skills
```

---

## 参考资料

- [Config | OpenCode](https://opencode.ai/docs/config/)
- [Agents | OpenCode](https://opencode.ai/docs/agents/)
- [Commands | OpenCode](https://opencode.ai/docs/commands/)
- [Rules | OpenCode](https://opencode.ai/docs/rules/)
- [Agent Skills | OpenCode](https://opencode.ai/docs/skills/)
- [Plugins | OpenCode](https://opencode.ai/docs/plugins/)
- [Custom Tools | OpenCode](https://opencode.ai/docs/custom-tools/)
