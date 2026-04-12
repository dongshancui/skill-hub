# Agent Harness 开发全流程实战指南

> 聚焦四个环节：需求分析 → 方案设计 → 开发实现 → 测试修复  
> 原则：每条建议都要能落地，拒绝空话。

---

## 环节一：需求分析

### 核心问题

需求分析环节，Agent 最容易犯的错误是**直接开始做**——拿到需求描述就开始想技术方案，跳过了理解需求本身。

人也会犯同样的错，但人有隐性的业务经验兜底。Agent 没有，所以需要显式地把"理解需求"做成一个独立步骤，强制输出结构化产物。

---

### 实践 1：需求输入标准化

**问题**：需求来自 Jira、口述、飞书文档、群聊截图，格式五花八门，Agent 读到什么都不确定。

**做法**：在所有需求进入 Agent 之前，先经过一个**需求标准化 prompt**，输出固定格式：

```markdown
## 需求标准化模板（Requirement Brief）

### 背景
[一句话：这个需求解决的是谁的什么问题]

### 触发场景
[用户/系统在什么情况下触发这个功能]

### 期望行为
[成功时，系统应该做什么；用列表写，每条一个行为]
- 
- 

### 边界条件
[什么情况不在本次范围内]

### 验收标准
[怎么判断做完了、做对了]
- AC1: 
- AC2: 

### 影响范围（初步）
[哪些模块/服务/接口可能需要改动，先填猜测，后面设计阶段核实]
```

**关键点**：这个模板不是让 Agent 填的，而是**让人类在输入需求时填**（或者用 Agent 辅助提取，但必须人工确认）。这是 Agent 后续所有工作的地基，不能含糊。

---

### 实践 2：需求歧义检测

**问题**：需求描述里有大量隐含假设和歧义，人类看惯了会自动脑补，Agent 会按字面意思理解。

**做法**：需求标准化之后，让 Agent 做一轮**歧义检测**，输出它不确定的问题清单：

```
Prompt 示例：

你是一名资深后端工程师。请仔细阅读下面的需求描述，
列出所有你认为存在歧义、缺少信息、或者隐含假设的地方。

每个问题用以下格式输出：
- [问题]：具体描述歧义在哪里
- [影响]：如果理解错了，会导致什么问题
- [建议]：你推测的可能答案是什么

需求描述：
{{requirement_brief}}
```

**典型输出示例**：

```
- [问题]：需求说"用户可以删除订单"，但没说删除后订单数据是物理删除还是软删除
- [影响]：如果做了物理删除，客服后续无法查询历史订单纠纷
- [建议]：推测应该是软删除（status 标记），需要产品确认

- [问题]："批量导出"没有说数据量上限
- [影响]：如果数据量很大，同步导出会超时；需要决定是同步还是异步
- [建议]：建议设置上限 10000 条，超出时异步导出并发邮件通知
```

这份清单交给产品/业务方确认，确认结果**追加到需求 Brief 里**，不是口头说完就算。

---

### 实践 3：多仓影响面分析

**问题**：一个需求可能涉及多个服务，但需求描述通常只说业务，不说哪些服务受影响。

**做法**：建立一个**服务依赖图谱文件**，Agent 读取后做影响面分析：

```markdown
# cross-repo-deps.md（维护在 memory/ 目录）

## 服务调用关系
- user-service → auth-service（用户登录/鉴权）
- order-service → user-service（查询用户信息）
- order-service → inventory-service（库存扣减）
- payment-service → order-service（支付回调更新订单）
- notification-service ← order-service（订单事件推送）

## 共享数据库
- user_db: user-service（写）, admin-service（只读）
- order_db: order-service（写）, report-service（只读）

## 外部依赖
- 短信: 阿里云 SMS，封装在 notification-service
- 支付: 微信支付、支付宝，封装在 payment-service
```

Agent 拿到需求 Brief + 这张图，输出：

```
本次需求涉及服务：
- order-service（主要改动）
- inventory-service（库存接口可能需要新增字段）
- notification-service（新增订单取消通知模板）

不涉及服务：
- payment-service、user-service、auth-service

需要跨团队沟通：
- inventory-service 属于仓储团队，需要确认接口变更排期
```

---

### 需求分析阶段产物清单

进入方案设计之前，必须有这三个文件：

```
/task-{id}/
  ├── requirement-brief.md    # 标准化需求（人工确认过）
  ├── ambiguity-resolved.md   # 歧义问题及确认结论
  └── impact-scope.md         # 影响服务范围
```

**没有这三个文件，不进入方案设计阶段。**

---

---

## 环节二：方案设计

### 核心问题

方案设计是 Agent 最容易"看起来做了很多，但实际上没有思考"的环节。Agent 会生成洋洋洒洒的文档，但可能只是在拼凑模板，没有真正分析当前代码的约束。

关键原则：**方案必须基于代码现状，不能凭空设计。**

---

### 实践 4：代码现状 Baseline 分析

**做法**：方案设计第一步，让 Agent 先读代码，输出现状描述，再设计方案。

```
Prompt 示例：

在给出方案之前，请先分析以下仓库的现状：
1. 找到与本次需求最相关的现有代码：[列出文件路径]
2. 描述当前的实现方式（数据结构、关键逻辑、接口定义）
3. 指出当前实现中与本次需求有关的约束或限制
4. 列出你需要额外了解才能设计方案的代码位置

需求简报：{{requirement_brief}}
相关仓库：{{repo_list}}
```

**Agent 输出示例**：

```markdown
## 代码现状分析

### 当前订单状态机（order-service/domain/OrderStatus.java:23）
现有状态：PENDING → PAID → SHIPPED → COMPLETED
                           ↓
                        CANCELLED（只能从 PENDING 取消）

### 约束发现
- 取消逻辑写死在 OrderService.cancel()，只判断 PENDING 状态
- 没有取消原因字段（cancel_reason），数据库表也没有这列
- inventory-service 的库存回退接口（/api/inventory/rollback）已存在

### 需要进一步确认的代码
- notification-service 是否有取消通知的模板（路径未知，需要搜索）
- order_db 的 orders 表结构（需要查看 migration 文件）
```

这份分析比方案本身更重要——它暴露了设计时必须考虑的约束。

---

### 实践 5：方案文档的强制结构

**问题**：Agent 生成的方案文档容易写成"计划书"，全是将来时态，缺少具体的技术决策。

**做法**：用固定模板约束方案文档结构，每个 section 都要求具体内容，不允许空话：

```markdown
## 技术方案模板（Solution Design）

### 方案概述
[一段话，说清楚用什么方式解决问题，不超过 100 字]

### 数据变更
- 新增表/字段（如有）：
  ```sql
  ALTER TABLE orders ADD COLUMN cancel_reason VARCHAR(200);
  ALTER TABLE orders ADD COLUMN cancelled_at DATETIME;
  ```
- 无数据变更：明确写"无"

### 接口变更
- 新增接口（如有）：方法 + 路径 + 请求体 + 响应体
- 修改接口（如有）：改了哪个字段，是否向后兼容
- 无接口变更：明确写"无"

### 核心逻辑变更
[用伪代码或流程图描述关键逻辑，不要用自然语言描述代码]

```
// 取消订单逻辑变更
cancelOrder(orderId, reason, operatorId):
  order = findById(orderId)
  if order.status not in [PENDING, PAID]:
    throw InvalidStatusException
  if order.status == PAID:
    triggerRefund(order)  // 新增：已支付订单退款
  order.status = CANCELLED
  order.cancelReason = reason  // 新增字段
  order.cancelledAt = now()
  rollbackInventory(order)
  sendCancelNotification(order)
```

### 影响的文件列表
[提前列出需要修改的文件，尽量精确到类名]
- order-service/src/.../OrderService.java（核心逻辑）
- order-service/src/.../OrderController.java（接口变更）
- order-service/src/db/migration/V20_add_cancel_fields.sql（数据变更）
- notification-service/src/.../templates/order-cancel.ftl（新增模板）

### 方案风险
[列出可能出问题的地方和规避方式]
- 风险 1：已有 PAID 订单取消需要触发退款，退款接口超时会导致取消失败 → 退款异步处理，取消先执行
- 风险 2：cancel_reason 字段长度可能不够 → 先设 500 字，超长截断

### 不在本次范围内
[明确写出边界，防止范围蔓延]
- 取消原因的统计报表（下期需求）
- 管理员强制取消（需要单独权限设计）
```

---

### 实践 6：方案评审 Checklist

方案写完后，用固定 checklist 做自检（可以让 Agent 执行，也可以人工）：

```markdown
## 方案评审 Checklist

### 完整性
- [ ] 数据变更是否有对应 migration 脚本
- [ ] 新增接口是否定义了错误码和错误响应
- [ ] 影响的文件列表是否完整（有没有漏掉的 DTO、Mapper、配置文件）
- [ ] 跨服务调用是否评估了服务不可用时的降级方案

### 一致性
- [ ] 新增字段命名是否符合现有代码风格
- [ ] 新增接口路径是否符合现有 URL 规范
- [ ] 错误处理方式是否与现有代码一致

### 安全性
- [ ] 新增接口是否需要鉴权
- [ ] 用户输入是否有长度/格式校验
- [ ] 是否涉及敏感数据（需要脱敏/加密）

### 可测试性
- [ ] 核心逻辑是否可以写单测（没有过多外部依赖）
- [ ] 边界条件（错误码、异常状态）是否可以通过测试验证
```

Checklist 没有全部通过，方案不进入开发。

---

### 方案设计阶段产物清单

```
/task-{id}/
  ├── requirement-brief.md      # （来自上一阶段）
  ├── ambiguity-resolved.md     # （来自上一阶段）
  ├── impact-scope.md           # （来自上一阶段）
  ├── code-baseline.md          # 代码现状分析（新增）
  ├── solution-design.md        # 技术方案（新增）
  └── solution-checklist.md     # 评审 checklist 结果（新增）
```

---

---

## 环节三：开发实现

### 核心问题

开发实现是 Agent 能力最强的环节，但也是出问题最多的环节。最常见的问题不是"写不出来"，而是：

1. **写了太多**：改了不该改的文件，"顺手"重构了周边代码
2. **写得不一致**：命名风格、错误处理方式与现有代码不符
3. **遗漏了**：改了 Service 但忘了改对应的 DTO 或 Mapper
4. **没有遵守约束**：绕过了 AGENTS.md 里写的规范

---

### 实践 7：任务分解到原子级别

**问题**：让 Agent 一次性实现整个需求，很容易出现遗漏、前后不一致。

**做法**：把方案里的"影响文件列表"拆成**有序的原子任务**，每个任务只改一个文件或一件事：

```markdown
## 开发任务分解（Task Breakdown）

基于 solution-design.md，分解为以下原子任务：

### 数据层
- [ ] T1: 新增 migration 脚本 V20_add_cancel_fields.sql
- [ ] T2: 在 Order 实体类添加 cancelReason、cancelledAt 字段
- [ ] T3: 更新 OrderMapper 的 ResultMap

### 业务层
- [ ] T4: 修改 OrderService.cancel() 支持 PAID 状态取消
- [ ] T5: 在 OrderService.cancel() 中添加异步退款调用
- [ ] T6: 在 OrderService.cancel() 中调用通知服务

### 接口层
- [ ] T7: 修改 CancelOrderRequest DTO，添加 cancelReason 字段
- [ ] T8: 修改 OrderController.cancel() 接口（参数变更）

### 跨服务
- [ ] T9: notification-service 新增取消通知模板

### 单测
- [ ] T10: 为 OrderService.cancel() 新增单测（覆盖 PAID 状态取消）
- [ ] T11: 为新接口参数校验写单测
```

**执行规则**：
- 按顺序执行，每完成一个任务就提交一次（小 commit）
- 每个任务完成后，Agent 列出本次修改的文件清单，人工对照预期
- 发现与预期不符，立即停止，不要继续下一个任务

---

### 实践 8：上下文聚焦，防止 Agent "乱逛"

**问题**：Agent 在实现 T4（修改 OrderService）时，可能"顺手"读了 UserService，然后发现可以优化，开始改不相关的东西。

**做法**：每个任务的 prompt 都明确指定**只允许读写的文件范围**：

```
Prompt 示例（T4 任务）：

你现在只需要完成以下任务，不要修改任何其他文件：

任务：修改 OrderService.cancel() 方法，支持 PAID 状态的订单取消

可以读取的文件（用于理解上下文）：
- order-service/src/main/java/com/xx/order/service/OrderService.java
- order-service/src/main/java/com/xx/order/domain/OrderStatus.java
- order-service/src/main/java/com/xx/order/domain/Order.java

需要修改的文件（只能改这一个）：
- order-service/src/main/java/com/xx/order/service/OrderService.java

方案参考：[粘贴 solution-design.md 中的核心逻辑伪代码]

约束：
- 不要重构方法命名
- 不要调整与本次无关的代码格式
- 不要添加任何注释（除非是关键逻辑说明）
```

这个约束看起来啰嗦，但实际上节省了大量的 review 时间。

---

### 实践 9：代码风格一致性

**问题**：Agent 写的代码和现有代码风格不一致，review 的人需要花时间"翻译"。

**做法**：在 AGENTS.md 里维护一个**代码风格示例库**，而不是用自然语言描述风格：

```markdown
# order-service/AGENTS.md 中的风格部分

## 代码风格（看代码，不看描述）

### 错误处理方式
参考现有代码：OrderService.java:87-95
```java
// 正确做法：抛业务异常，不要 return null
if (order == null) {
    throw new OrderNotFoundException(orderId);
}
// 错误做法：不要这样写
if (order == null) {
    return Result.fail("order not found");
}
```

### 日志规范
```java
// 正确：用占位符，不要字符串拼接
log.info("取消订单, orderId={}, operator={}", orderId, operatorId);
// 错误：不要这样写
log.info("取消订单" + orderId);
```

### Service 方法签名规范
```java
// 参数超过 3 个时，封装成 Command 对象
public void cancelOrder(CancelOrderCommand command) {...}
// 不要这样写
public void cancelOrder(Long orderId, String reason, Long operatorId, ...) {...}
```
```

**关键**：给 Agent 看真实的代码例子，比说"遵守现有风格"有效 10 倍。

---

### 实践 10：实现过程的 Diff 控制

每个原子任务完成后，让 Agent 输出一个简短的**变更摘要**，你对照 checklist 确认：

```markdown
## T4 变更摘要

修改文件：OrderService.java

变更内容：
1. cancel() 方法新增对 PAID 状态的处理（第 87-102 行）
2. 新增私有方法 triggerAsyncRefund()（第 156-171 行）
3. 调用已有的 rollbackInventory() 方法（第 98 行）

未修改内容（确认）：
- 方法签名未变（向后兼容）
- 其他方法未改动

潜在问题（已处理）：
- triggerAsyncRefund 中的异常捕获，防止退款失败阻断取消流程
```

这个摘要不是给 Agent 看的，是给**你**看的，判断这个任务是否可以进入下一步。

---

### 实践 11：多仓开发的顺序原则

当需求跨多个仓库时，有固定的开发顺序：

```
正确顺序：
  1. 先改数据层（migration 脚本）
  2. 再改被调用方（下游服务的接口/事件）
  3. 再改调用方（上游服务的逻辑）
  4. 最后改接入层（Controller、API 网关）

错误做法：
  - 同时改多个仓库，中间状态不可运行
  - 先改调用方，被调用方还没改，测试无法进行
```

每个仓库的改动单独提交，提交信息格式统一：

```
feat(order-service): 支持 PAID 状态订单取消 [task-123]

- 修改 OrderService.cancel() 支持 PAID → CANCELLED 状态流转
- 新增 cancelReason、cancelledAt 字段（对应 migration V20）
- 退款触发改为异步，防止支付服务超时影响取消流程
```

---

### 实践 12：开发过程的暂停点

以下情况，Agent 必须停下来，等人工确认后继续：

```markdown
## 强制暂停场景

1. 发现方案中没有覆盖的情况
   例：发现还有一种 REFUNDING 状态，方案里没写怎么处理
   → 停下，补充方案，不要自行决定

2. 需要修改方案之外的文件
   例：为了实现功能，发现需要改一个工具类
   → 停下，说明原因，人工确认后继续

3. 发现现有代码有 Bug（和本次需求无关）
   例：实现过程中发现 findById 没有做空指针保护
   → 记录下来（写入 todo.md），本次不改，不要"顺手修"

4. 不确定某个边界条件的处理方式
   例：不知道取消原因是否允许为空
   → 停下，问，不要自己猜
```

---

---

## 环节四：测试修复

### 核心问题

测试修复环节，Agent 的常见问题是：

1. **测试写得太浅**：只测 happy path，不测边界和异常
2. **测试与实现耦合**：测试里写死了实现细节，一重构就全挂
3. **修复方式治标不治本**：测试失败了，改测试而不是改代码
4. **循环修复**：改了 A，B 又挂了，改了 B，C 又挂了

---

### 实践 13：测试用例设计前置

**做法**：不要等代码写完再想测什么。在方案设计阶段，就同步输出**测试用例大纲**：

```markdown
## 测试用例大纲（Test Case Outline）

### OrderService.cancel() 测试

#### Happy Path
- TC1: PENDING 状态订单取消成功 → 状态变为 CANCELLED，库存回退，通知发送
- TC2: PAID 状态订单取消成功 → 状态变为 CANCELLED，触发退款，库存回退，通知发送

#### 异常状态
- TC3: SHIPPED 状态订单取消 → 抛 InvalidStatusException，订单不变
- TC4: COMPLETED 状态订单取消 → 抛 InvalidStatusException，订单不变
- TC5: 订单不存在 → 抛 OrderNotFoundException

#### 边界条件
- TC6: cancelReason 为 null → 允许，字段存 null
- TC7: cancelReason 超过 500 字 → 截断为 500 字存储
- TC8: 退款服务超时 → 取消成功，退款进入异步重试队列，不抛异常

#### 并发场景
- TC9: 同一订单并发取消两次 → 只有一次成功，另一次抛异常（幂等）
```

这份大纲写完，实际上你已经对需求的理解深了一个层次——很多边界条件是在写测试用例时才想清楚的。

---

### 实践 14：单测结构标准化（AAA 模式）

让 Agent 严格按照 Arrange-Act-Assert 结构写单测：

```java
@Test
void cancelOrder_whenStatusIsPaid_shouldCancelAndTriggerRefund() {
    // Arrange（准备：构造测试数据，打桩外部依赖）
    Long orderId = 1L;
    Order order = Order.builder()
        .id(orderId)
        .status(OrderStatus.PAID)
        .totalAmount(new BigDecimal("100.00"))
        .build();
    when(orderRepository.findById(orderId)).thenReturn(Optional.of(order));
    when(refundService.triggerRefundAsync(any())).thenReturn(CompletableFuture.completedFuture(null));

    // Act（执行：调用被测方法）
    orderService.cancelOrder(new CancelOrderCommand(orderId, "用户申请取消", 99L));

    // Assert（验证：检查结果和副作用）
    assertThat(order.getStatus()).isEqualTo(OrderStatus.CANCELLED);
    assertThat(order.getCancelReason()).isEqualTo("用户申请取消");
    assertThat(order.getCancelledAt()).isNotNull();
    verify(refundService).triggerRefundAsync(any(RefundCommand.class));
    verify(inventoryService).rollback(any(InventoryRollbackCommand.class));
    verify(notificationService).sendOrderCancelledNotification(orderId);
}
```

**关键约束**（写在 AGENTS.md）：
- 测试方法命名：`被测方法_场景描述_期望结果`
- 每个测试只 assert 一件主要的事
- Mock 只打桩直接依赖，不要 mock 被测类自身的方法
- 不允许在测试里写业务逻辑（if/for）

---

### 实践 15：测试失败的分析流程

测试挂了，不要让 Agent 直接改代码。先做原因分析：

```
Prompt 示例：

以下测试失败了，请分析失败原因，不要直接给出修复代码。

测试名称：cancelOrder_whenStatusIsPaid_shouldCancelAndTriggerRefund
失败信息：
  Expected: CANCELLED
  Actual:   PAID
  at OrderServiceTest.java:87

请回答：
1. 失败的根本原因是什么（代码逻辑问题？测试数据问题？Mock 设置问题？）
2. 失败的是代码的问题还是测试的问题？
3. 如果是代码问题，哪行代码需要修改，为什么？
4. 如果是测试问题，测试哪里写错了？
```

这个分析步骤强制 Agent（和你自己）在改代码之前先想清楚。避免"试试这样改行不行"的盲目修复。

---

### 实践 16：禁止修改测试来让测试通过

这是最重要的一条原则，必须写入 AGENTS.md：

```markdown
# AGENTS.md 中的测试原则

## 测试修复规则

**严禁**通过以下方式让测试通过：
- 修改测试的 Assert 期望值（除非期望值本身写错了）
- 删除失败的测试用例
- 给测试方法加 @Ignore / @Disabled
- 修改 Mock 来掩盖真实的调用失败

**正确的修复方向**：
- 测试的 Arrange 有问题（构造数据不对）→ 修测试
- 测试的 Assert 理解需求有误 → 先确认需求，再修测试
- 被测代码逻辑有 Bug → 修代码
- 外部依赖行为与预期不符 → 检查 Mock 是否正确模拟了真实行为
```

---

### 实践 17：失败测试的根因分类

Agent 修复失败测试时，先做分类，再做决策：

```markdown
## 测试失败根因分类

### A 类：代码 Bug（修代码）
- 状态流转逻辑错误
- 字段赋值遗漏
- 调用顺序错误
- 并发处理缺失

### B 类：测试设计问题（修测试）
- Mock 打桩不完整，导致 NPE
- 测试数据构造不符合业务约束
- Assert 写的是实现细节而不是业务行为

### C 类：需求理解偏差（回到需求确认）
- 测试期望的行为与产品需求不符
- 边界条件没有在需求阶段澄清
→ 必须回到需求文档确认，不能凭猜测修复

### D 类：环境问题（修环境）
- 测试数据库连接问题
- 外部服务 Mock 配置缺失
- 依赖版本冲突
```

每次修复前，先判断是哪类问题，对应找正确的修复路径。

---

### 实践 18：修复后的回归检查

每次修复一个失败测试后，不要只运行这一个测试，而是运行**整个模块的测试**：

```bash
# 不要只运行单个测试
mvn test -Dtest=OrderServiceTest#cancelOrder_whenStatusIsPaid

# 要运行整个 Service 层测试
mvn test -Dtest=*ServiceTest

# 修复数据层相关问题后，运行所有测试
mvn test
```

并且要求 Agent 在提交修复前，输出测试运行摘要：

```
测试运行结果：
- 总计：124 个测试
- 通过：124 个
- 失败：0 个
- 跳过：2 个（已存在的 @Disabled，与本次无关）

新增测试：11 个（覆盖 TC1-TC9 + 2 个边界条件）
覆盖率变化：OrderService 从 76% → 89%
```

---

### 实践 19：测试覆盖的优先级

不要追求 100% 覆盖率，而是覆盖**对的地方**：

```markdown
## 测试覆盖优先级

### 必须测（P0）
- 核心业务逻辑（状态机、金额计算、权限判断）
- 所有异常分支（抛出的每种业务异常）
- 外部服务调用失败的降级逻辑

### 应该测（P1）
- 边界输入（null、空字符串、超长、0、负数）
- 并发场景（幂等性）
- 数据变更（数据库写入的字段是否正确）

### 可以不测（P2）
- 简单的 getter/setter
- 纯粹的数据格式转换（DTO 转 Entity）
- 框架提供的功能（Spring 的依赖注入、参数校验注解）
```

---

## 各环节产物总览

```
/task-{id}/
  ├── 需求分析阶段
  │   ├── requirement-brief.md       # 标准化需求（人工确认）
  │   ├── ambiguity-resolved.md      # 歧义澄清记录
  │   └── impact-scope.md            # 影响服务范围
  │
  ├── 方案设计阶段
  │   ├── code-baseline.md           # 代码现状分析
  │   ├── solution-design.md         # 技术方案
  │   ├── solution-checklist.md      # 方案评审结果
  │   └── test-case-outline.md       # 测试用例大纲（同步产出）
  │
  ├── 开发实现阶段
  │   ├── task-breakdown.md          # 原子任务分解清单
  │   └── todo.md                    # 发现的无关 Bug（本次不改）
  │
  └── 测试修复阶段
      ├── test-failure-analysis.md   # 失败根因分析记录
      └── test-run-summary.md        # 最终测试运行结果
```

---

## 跨环节的通用原则

### 原则 1：每个环节都有人工确认点

```
需求分析 → [人工确认需求 Brief + 歧义澄清]
    ↓
方案设计 → [人工确认技术方案 + Checklist]
    ↓
任务分解 → [人工确认原子任务列表]
    ↓
每个原子任务完成 → [人工确认变更摘要]
    ↓
测试全部通过 → [人工查看覆盖率 + 关键测试用例]
```

每个确认点都是**检查点，不是橡皮图章**。如果你只是看了一眼就点确认，这套流程的价值减半。

### 原则 2：Agent 发现的问题必须显式记录

Agent 在任何环节发现的问题，无论是否与当前任务相关，都要写入对应文档：

- 与当前任务相关 → 在当前方案/任务中处理
- 与当前任务无关的 Bug → 写入 todo.md，等待下次处理
- 与当前任务无关的设计问题 → 写入 todo.md，可转 Jira

**不允许 Agent 自行判断"这个问题顺手改一下"。**

### 原则 3：每次 commit 对应一个原子任务

不要攒完所有改动再提交。原子任务粒度提交有三个好处：

1. review 每次只看一件事，注意力集中
2. 某个任务出问题，可以精确回滚
3. commit 历史就是任务进度记录

### 原则 4：文档和代码同步更新

当代码因为新发现的约束需要调整时，方案文档同步更新。不允许代码和方案文档脱节。三个月后你（或者其他人）回来看这个任务，文档必须能还原当时的决策过程。
