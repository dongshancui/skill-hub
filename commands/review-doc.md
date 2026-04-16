---
description: 启动 subagent 对文档进行循环 review，直到满足用户要求
---

从当前会话上下文中确定目标文档路径。$ARGUMENTS 为 review 要求。

启动一个 subagent 执行以下循环：

1. 读取文档全部内容
2. 对照 review 要求逐条检查，判断是否满足
3. 若全部通过，输出"review 通过"并结束循环
4. 若存在问题，列出具体问题及修改方案，直接修复文档
5. 修复完成后回到第 1 步重新 review

subagent 不询问确认，发现问题直接修复，修复后继续 review，直到所有要求均满足为止。
