---
description: 启动 subagent 对文档进行循环 review，直到满足用户要求
---

从当前会话上下文中确定目标文档路径。$ARGUMENTS 为 review 要求。

按以下循环执行，直到 review 通过：

1. **启动 subagent 执行 review**：读取文档全部内容，对照 review 要求逐条检查，返回结果：通过或列出具体问题
2. **主会话判断结果**：
   - 若 subagent 返回通过，输出"review 通过"并结束
   - 若存在问题，主会话根据 subagent 的问题列表修复文档
3. 修复完成后回到第 1 步，重新启动 subagent review

subagent 只负责审查和报告，不修改文件。修复由主会话执行。
