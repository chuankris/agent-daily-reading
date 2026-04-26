# 12 从 Spring AI 看 Java 生态里的 MCP

原文来源：

- Spring AI MCP Overview
- 链接：https://docs.spring.io/spring-ai/reference/api/mcp/mcp-overview.html
- 辅助来源：https://docs.spring.io/spring-ai/reference/guides/getting-started-mcp.html

为什么今天读这个：

前面你已经理解了 MCP 的协议意义。现在这篇帮助你把它落回自己熟悉的 Java 世界，避免把 MCP 理解成“只属于 Python Agent 框架的东西”。

中文精读：

Spring AI 官方对 MCP 的描述很直白：

1. MCP 是 AI 模型和外部工具/资源交互的标准化桥梁
2. Java 生态也可以把 MCP 做成正式的 client/server 能力
3. Spring AI 提供了 Boot Starter 和注解化支持，帮助你更快接入

这意味着什么：

1. 你以后完全可以用 Java 写 MCP Server
- 把企业内部能力标准化暴露出来

2. 也可以用 Java 写 MCP Client
- 去消费别的 MCP Server 提供的资源和工具

3. 这和你原来的系统集成经验高度一致
- 只是协议对象从 REST/消息队列/SDK，扩展到了 Agent 世界

文档里特别值得你注意的点：

1. Transport 多样
- 不同环境下可以有不同传输方式

2. Capability negotiation
- client 和 server 要先协商彼此支持什么

3. Tool / Resource / Prompt 都是正式能力对象
- 不是只剩下一个“工具函数”

4. Spring AI 对 MCP 做了 Boot Starter 化
- 这对你非常友好，因为你不需要一上来就走底层 SDK

你只需要记住的 3 个点：

1. MCP 在 Java 生态里已经有正式落地路径，不是旁观者技术。
2. 你可以把企业内部能力封装成 MCP Server，这是很强的差异化方向。
3. 你过去做系统集成的经验，能直接迁移到 MCP 接入层设计。

和 Agent 工程的关系：

这篇会让你后面做项目时更有“组合拳”思维：Python 负责快速 Agent 原型，Java/Spring 负责企业能力沉淀和标准化对接。

今晚可动手：

列 3 个你以前项目里最适合暴露成 MCP Tool 的内部能力，例如：

- 设备状态查询
- 工单创建
- 告警确认
