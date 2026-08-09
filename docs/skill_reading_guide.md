# Skill 阅读导图

基于 [skill_verbatim.md](./skill_verbatim.md) 重组，按“阅读建议”拆成面向不同读者的模块化阅读文档。

## 怎么用

- 想快速上手：先看“模块 A”
- 想提升已有 Skill 质量：先看“模块 B”
- 想做团队规范和管理：先看“模块 C”
- 想接外部服务：先看“模块 D”
- Skill 跑不通：先看“模块 E”

---

## 模块 A：从零开始写出第一个 Skill

### 适合谁

- 从没写过 Skill
- 想先快速理解 Skill 是什么
- 想在最短时间内做出一个能跑的版本

### 推荐路径

1. 一、先搞清楚 Skill 是什么
2. 二、Skill 长什么样
3. 二.5 Quick Start
4. 三、写出好 Skill 的关键技巧
5. 八、用 Skill Creator 帮你写 Skill

### 该模块解决的问题

- Skill 本质是什么，和普通 Prompt、Rule 有什么区别
- 一个 Skill 目录里到底该放什么
- `SKILL.md` 该怎么写
- 第一个 Skill 怎么 5 分钟起步
- 如果不想手写，怎么用 Skill Creator 起草

### 重点章节

- [1.1 Skill 到底是啥](./skill_verbatim.md#skill-what-is)
- [1.3 Skill 怎么运作的](./skill_verbatim.md#skill-how-it-works)
- [2.1 最基本的样子](./skill_verbatim.md#skill-structure-basic)
- [2.2 SKILL.md 里面写什么](./skill_verbatim.md#skill-skillmd-content)
- [2.4 Skill 和 Rule 有什么区别](./skill_verbatim.md#skill-vs-rule)
- [2.5 Quick Start](./skill_verbatim.md#skill-quick-start)
- [3.1 Description 写好了，一切就成功了一半](./skill_verbatim.md#skill-description)
- [8.1 怎么装](./skill_verbatim.md#skill-creator-install)
- [8.2 核心工作流程](./skill_verbatim.md#skill-creator-workflow)

### 阅读后的产出目标

- 你能独立搭一个最小 Skill 目录
- 你能写出基础版 `name` 和 `description`
- 你能生成一个能触发的入门 Skill

---

## 模块 B：把 Skill 从“能用”提升到“好用”

### 适合谁

- 写过 Skill，但效果不稳定
- AI 经常触发不准、执行跑偏
- 想系统提升 Skill 的可读性、可执行性和稳定性

### 推荐路径

1. 三、写出好 Skill 的关键技巧
2. 五、一些进阶的写法
3. 十二、常见反模式
4. 十三、总结检查清单

### 该模块解决的问题

- 怎么写出更容易触发的 description
- 怎么写步骤、示例、Few-Shot、Before/After
- 怎么用表格、脚本、分方案、FAQ 提高稳定性
- 哪些反模式最常导致“写了但不好使”
- 最后如何自查质量

### 重点章节

- [3.1 Description 写好了，一切就成功了一半](./skill_verbatim.md#skill-description)
- [3.2 开头就说清楚：做什么、为什么、要不要做](./skill_verbatim.md#skill-goal-and-scope)
- [3.4 给出改之前 vs 改之后的对比](./skill_verbatim.md#skill-before-after)
- [3.5 Few-Shot，多给几个例子](./skill_verbatim.md#skill-few-shot)
- [3.6 善用可视化：决策树与流程图](./skill_verbatim.md#skill-visualization)
- [5.1 能用表格就用表格](./skill_verbatim.md#skill-use-tables)
- [5.2 复杂检查逻辑？写成脚本](./skill_verbatim.md#skill-use-scripts)
- [5.3 提供多种方案适配不同场景和项目](./skill_verbatim.md#skill-multiple-strategies)
- [5.5 FAQ 不是摆设](./skill_verbatim.md#skill-faq)
- [十二、别踩这些坑](./skill_verbatim.md#skill-antipatterns)
- [十三、总结：写好 Skill 的核心检查清单](./skill_verbatim.md#skill-final-checklist)

### 阅读后的产出目标

- 你能系统识别自己 Skill 的薄弱点
- 你能把“描述型 Skill”升级成“示例驱动型 Skill”
- 你能建立自己的 Skill 质量检查清单

---

## 模块 C：团队规范、模块化与生命周期管理

### 适合谁

- 负责团队 Skill 沉淀
- 需要做共享规范、复用、版本管理
- 关心长期维护、协作和工程结构

### 推荐路径

1. 四、Skill 太长了怎么办
2. 七、安全意识
3. 十一、Skill 的生命周期管理
4. 十二、常见反模式

### 该模块解决的问题

- 什么时候应该拆 Skill
- 主 Skill / 子 Skill 怎么编排
- 团队共享时如何做版本、复用、协作
- 如何把安全和生命周期纳入 Skill 管理

### 重点章节

- [4.1 什么时候该拆](./skill_verbatim.md#skill-when-to-split)
- [4.2 拆成什么样](./skill_verbatim.md#skill-modular-design)
- [4.3 主 Skill 如何编排子 Skill](./skill_verbatim.md#skill-orchestration)
- [4.4 拆分的几个原则](./skill_verbatim.md#skill-split-principles)
- [7.1 绝不硬编码敏感信息](./skill_verbatim.md#skill-security-secrets)
- [7.2 危险操作必须加确认](./skill_verbatim.md#skill-security-dangerous-actions)
- [7.5 安全检查清单](./skill_verbatim.md#skill-security-checklist)
- [11.1 版本控制](./skill_verbatim.md#skill-lifecycle-versioning)
- [11.2 跨项目复用](./skill_verbatim.md#skill-lifecycle-reuse)
- [11.3 团队协作与分享](./skill_verbatim.md#skill-lifecycle-collaboration)
- [11.4 持续迭代优化](./skill_verbatim.md#skill-lifecycle-iteration)

### 阅读后的产出目标

- 你能设计适合团队的 Skill 目录和拆分边界
- 你能给 Skill 增加安全底线和版本策略
- 你能把 Skill 当成长期维护的工程资产来管理

---

## 模块 D：外部服务接入与工具协同

### 适合谁

- 想让 Skill 操作外部系统
- 需要在 MCP、脚本、HTTP/API 之间做方案选择
- 关心鉴权、复用性和自动化集成

### 推荐路径

1. 六、要调外部服务？MCP vs HTTP
2. 七、安全意识

### 该模块解决的问题

- MCP 和直接 HTTP 调用的差异是什么
- 什么情况优先 MCP，什么情况直接脚本更合理
- MCP + HTTP 混用时怎么拆责任
- 外部调用的安全边界在哪里

### 重点章节

- [6.1 它们的区别在哪](./skill_verbatim.md#skill-mcp-vs-http-diff)
- [6.2 怎么选：跟着这个思路走](./skill_verbatim.md#skill-mcp-vs-http-choice)
- [6.3 场景一：优先用 MCP](./skill_verbatim.md#skill-use-mcp)
- [6.4 场景二：直接 HTTP 调也行](./skill_verbatim.md#skill-use-http)
- [6.5 场景三：MCP + HTTP 混着用](./skill_verbatim.md#skill-use-mcp-http-mixed)
- [6.6 避坑提示](./skill_verbatim.md#skill-mcp-http-pitfalls)
- [7.1 绝不硬编码敏感信息](./skill_verbatim.md#skill-security-secrets)
- [7.4 防范 Prompt 注入](./skill_verbatim.md#skill-security-prompt-injection)

### 阅读后的产出目标

- 你能判断一个外部能力应该做成 MCP 还是脚本
- 你能给外部调用路径加上基本安全防护
- 你能写出带鉴权、超时、校验、回退的调用说明

---

## 模块 E：验证、排障与调优

### 适合谁

- Skill 已经写了，但经常不触发或执行偏了
- 想建立验证和调试闭环
- 想用测试、评估和排错方法提升稳定性

### 推荐路径

1. 九、做完了怎么验证
2. 十、Skill 跑不通？调试与排错指南
3. 八.4 工程化评估

### 该模块解决的问题

- Skill 写完之后怎么做验证
- 怎么配检查清单和可复制的验证命令
- 触发不准、冲突、执行偏、脚本报错怎么查
- 怎么用评估数据反向改进 Skill

### 重点章节

- [8.4 工程化评估：让 Skill 质量有数据可依](./skill_verbatim.md#skill-creator-evaluation)
- [9.1 列清单验证](./skill_verbatim.md#skill-validation-checklist)
- [9.2 提供验证命令](./skill_verbatim.md#skill-validation-commands)
- [9.3 怎么评估 Skill 好不好用](./skill_verbatim.md#skill-validation-evaluation)
- [10.1 AI 该触发 Skill 却没触发](./skill_verbatim.md#skill-debug-not-triggered)
- [10.2 多个 Skill 冲突了怎么办](./skill_verbatim.md#skill-debug-conflicts)
- [10.3 AI 触发了 Skill 但执行偏了](./skill_verbatim.md#skill-debug-execution-drift)
- [10.4 脚本执行报错](./skill_verbatim.md#skill-debug-script-errors)
- [10.5 调试的一般方法论](./skill_verbatim.md#skill-debug-methodology)

### 阅读后的产出目标

- 你能给 Skill 配出一套最小验证闭环
- 你能快速定位“没触发 / 触发错 / 执行偏 / 脚本错”是哪一类问题
- 你能把评估结果反馈回 Skill 设计

---

## 横向专题索引

### 如果你只想看“写法”

- 三、写出好 Skill 的关键技巧
- 五、一些进阶的写法
- 十二、常见反模式

### 如果你只想看“工程化”

- 四、模块化拆分
- 七、安全意识
- 八.4 工程化评估
- 九、验证
- 十一、生命周期管理

### 如果你只想看“排错”

- 九、做完了怎么验证
- 十、调试与排错指南

### 如果你只想看“管理视角”

- 四、模块化
- 七、安全
- 十一、生命周期
- 十三、总检查清单

---
