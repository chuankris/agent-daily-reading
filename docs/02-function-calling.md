# 02 Function Calling 是什么

原文来源：

- OpenAI Function Calling Guide
- 链接：https://platform.openai.com/docs/guides/function-calling/introduction

为什么今天读这个：

Agent 工程最基础的动作，不是“聊天”，而是“让模型提出一个结构化调用请求，然后由你的系统真正执行工具”。

中文精读：

Function Calling 的关键，不是模型真的去执行函数，而是模型根据你给出的工具定义，产出一个结构化的调用意图。

系统真正要做的是：

1. 定义工具
- 给出工具名
- 给出参数结构
- 给出参数说明

2. 让模型挑选工具
- 模型返回的不是普通文本，而是工具名和参数

3. 应用侧执行工具
- 真正去查数据库、访问接口、读文件、调用服务的是你的代码，不是模型

4. 把结果再喂回模型
- 模型根据工具结果继续回答，或者继续下一步

这里最容易误解的地方有两个：

1. 模型不会替你运行函数
- 模型只负责“决定调用什么”和“传什么参数”

2. Function Calling 不是万能的 Agent
- 它只是 Agent 最基本的执行接口之一
- 你还需要重试、鉴权、超时、幂等、错误处理、日志

你只需要记住的 3 个点：

1. Function Calling 的本质是“结构化调用意图生成”。
2. 真正执行工具的是你的应用层。
3. 工具 schema 写得越清楚，模型越不容易乱调。

和你的转型关系：

你做 IoT 集成时，本质上就是系统和系统之间的动作编排。Function Calling 只是把“动作入口”从人工写死，变成由模型先给出结构化决策。

今晚可动手：

用 JSON 写一个 `create_ticket` 工具 schema，至少包含：

- title
- priority
- assignee
- description
