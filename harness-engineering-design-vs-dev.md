# Harness Engineering：设计阶段 vs 开发阶段的文档分工

---

## 结论：方式 B 更合理，但"关键 SQL"的边界要划清楚

**方式 A 的根本问题**：设计阶段把所有 SQL 和代码写完，等于把开发工作做了两遍——设计文档里写一遍，agent 再实现一遍。而且两份内容一旦不同步，设计文档就变成了负担而不是约束。

业界的核心原则来自 Red Hat 和多个 harness 实践：

> **"Enforce constraints, not methods. Define what must be true, not how to achieve it."**

---

## 两个阶段的正确分工

### 设计阶段：定义"什么是正确的"

应该包含的内容：

| 内容 | 为什么要放这里 | 粒度 |
|------|-------------|------|
| 数据库 schema（表/字段/索引/约束） | 架构决策，改了代价大，必须人工确认 | 完整 DDL |
| 核心业务流程图 | 定义 agent 工作的边界和顺序 | 流程节点，不是代码 |
| 复杂业务 SQL | 编码业务规则（同步逻辑、聚合、边界条件），这是**业务意图的精确表达**，不是实现细节 | 关键查询，不是 CRUD |
| 验收标准（AC） | 给 Evaluation Agent 用，agent 自己验证自己 | 具体可测 |
| 架构约束与禁止模式 | 等同于 AGENTS.md 里的规则 | 明确的 DO/DON'T |
| 关键边界条件和异常场景 | 防止 agent 在 happy path 上停下来 | 列举式 |

不应该包含的内容：
- CRUD 的实现代码（agent 能生成）
- 错误处理代码（agent 根据约束自己写）
- 测试代码（agent 根据 AC 自己写）
- API 脚手架（纯机械工作）

### 开发阶段：agent 自主实现 + 人工卡点验证

```
设计文档（输入）
    ↓
任务拆分（每个任务单一 scope，独立上下文）
    ↓
Agent 实现 → 运行测试 → 自我修复（循环）
    ↓
人工 Review Checkpoint（关键里程碑）
    ↓
对照 AC 验收
```

---

## "关键 SQL"怎么界定

这是方式 B 里最容易踩的坑。区分标准：

**放进设计文档的 SQL**：编码的是**业务规则**，写错了逻辑就错了
```sql
-- 同步逻辑：判断哪些记录需要更新（业务边界条件）
SELECT a.id, a.updated_at
FROM source_table a
LEFT JOIN target_table b ON a.id = b.source_id
WHERE b.id IS NULL
   OR a.updated_at > b.synced_at
   OR a.status != b.status
```

**不放进设计文档的 SQL**：纯机械实现，agent 能推导出来
```sql
-- 这种让 agent 自己写
INSERT INTO users (name, email, created_at) VALUES (?, ?, NOW())
SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC
```

判断标准就一条：**这条 SQL 包含的逻辑，换一个人写会不会写出不同的结果？** 会的话就放进设计文档；不会的话交给 agent。

---

## 对应到 Anthropic 的三 Agent 模型

设计和开发两个阶段，本质上对应：

```
你（人）          ← 写设计文档，相当于 Planning Agent 的输入
Planning Agent   ← 把设计文档拆成结构化任务（JSON feature specs）
Generation Agent ← 实现代码
Evaluation Agent ← 对照 AC 验证（Playwright 跑、测试跑、人工 review）
```

设计文档的精度决定 Planning Agent 能不能正确拆任务。设计文档太粗，agent 会做错架构决策；设计文档太细（写到所有代码），agent 的价值就消失了。

---

## 一句话总结

设计文档管的是**约束空间**，不是**实现路径**。数据库结构、业务规则 SQL、验收标准必须人工确认；代码实现、测试、错误处理交给 agent。
