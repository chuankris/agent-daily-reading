# 2026-05-20 Schema First：用 OpenAPI/JSON Schema 先收紧 Agent 工具边界

原文来源：

- 标题：OpenAPI Specification  
- 链接：https://swagger.io/specification/  
- 来源类型：官方规范

- 标题：JSON Schema  
- 链接：https://json-schema.org/  
- 来源类型：官方规范

为什么今天读这个：

你已经在做 Tool Calling / MCP 相关学习了。下一步最值钱的不是“再多接一个工具”，而是把工具契约固定下来，让模型输出和后端执行边界可控。

原文翻译（先读这一段）：

OpenAPI 和 JSON Schema 都强调机器可读的接口契约：字段类型、必填约束、枚举范围、嵌套结构都应显式声明。契约的价值不只是文档，而是为校验、代码生成、测试和兼容性治理提供统一依据。

中文精读：

对 Agent 工程来说，“Schema First” 的价值是把不确定性前置处理：

1. 调工具前校验参数  
- 不让明显无效请求进入业务系统

2. 调工具后校验结果  
- 防止下游消费脏数据

3. 版本变更有轨迹  
- 字段新增/删除时可明确影响面

推荐你用的最小流程：

1. 先写工具输入 Schema（最小字段集合）  
2. 写工具输出 Schema（下游真正会用的字段）  
3. 把校验集成进调用层（调用前后双校验）  
4. 用 5-10 个失败样例做回归测试

常见坑：

1. Schema 太宽松  
- `additionalProperties` 不收敛会导致不可预期字段渗透

2. 契约只写不验  
- 只有文档没有运行时校验，等于没约束

3. 版本变更无策略  
- 改字段后未区分向后兼容与破坏性变更

你只需要记住的 3 个点：

1. Schema 是 Agent 系统的“防漂移护栏”。  
2. 最关键是运行时校验，不是文档展示。  
3. 输入输出都要有契约，且都要进回归测试。

和 Agent 工程的关系：

这正是你的强项迁移点：你过去做企业集成时就重接口契约。把这套经验迁移到 Tool Calling/MCP，你会比纯 Prompt 路线更快进入生产能力层。

今晚可动手：

给你项目里一个工具定义输入输出 Schema，并补两类测试：

1. 缺失必填字段应失败  
2. 多余非法字段应拒绝（或明确策略）

原文关键段落翻译（人工翻译，放在文末）：

1. OpenAPI 规范提供了统一方式描述 HTTP API 的请求、响应和组件模型。  
2. JSON Schema 用于定义 JSON 文档结构与约束，并支持自动化校验。  
3. 把契约机器化，能显著提升接口协作效率和系统稳定性。

原文中文翻译链接（机器翻译）：

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://swagger.io/specification/
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://json-schema.org/
