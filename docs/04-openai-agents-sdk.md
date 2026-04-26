# 04 从 OpenAI Agents SDK 看 Agent 工程对象

原文来源：

- OpenAI Agents SDK
- 链接：https://developers.openai.com/api/docs/guides/agents

为什么今天读这个：

前面你会了概念，但还需要知道一个工程视角里的 Agent 到底由哪些对象组成。

中文精读：

Agents SDK 给你的不是一个“更聪明的聊天接口”，而是一套代码优先的 Agent 运行框架。

它强调几个工程边界：

1. 什么时候直接用普通 API client
- 只需要模型请求时

2. 什么时候用 Agents SDK
- 当你的服务自己负责编排、状态、工具执行、审批、可观察性时

3. 什么时候才用可视化 Agent Builder
- 当你明确想走托管工作流和嵌入式 UI 路线时

从阅读顺序看，这套文档在提醒你：

1. 先跑通一个 agent run
2. 再定义单个 specialist agent
3. 再理解 runtime loop 和 state
4. 再进入 handoff、guardrail、human review
5. 最后做 tracing、observability、eval

这其实很符合生产系统成长路径：

先闭环，再分工，再治理。

其中有几个关键词你现在就该熟：

1. Handoff
- 一个 agent 把任务转交给另一个更合适的 agent

2. Guardrail
- 在危险动作前做约束、校验、拦截

3. Results and state
- agent 不是只返回一句话，还会返回运行结果、状态、后续入口

4. Observability
- 不是锦上添花，而是调试 agent 的基础设施

你只需要记住的 3 个点：

1. Agents SDK 面向的是“自己掌控编排和运行时”的工程场景。
2. 一个像样的 Agent 系统，天然包含状态、工具、审批、handoff、trace。
3. 可观测性和评测不是后补的，而是从一开始就该考虑。

和你的转型关系：

这正好对应你从“Java 后端工程师”到“Agent 工程师”的桥。因为你真正有优势的，不是玩 Prompt，而是能把运行时、状态、异常、审批链路做稳。

今晚可动手：

写一张纸，列出你理想中的最小 Agent 服务需要哪些对象：

- task
- tool
- trace
- result
- state
