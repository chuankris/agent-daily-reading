# 10 为什么 Agent 更需要可观测性

原文来源：

- Spring AI Observability
- 链接：https://docs.spring.io/spring-ai/reference/observability/index.html

为什么今天读这个：

普通后端出错，你看接口日志、SQL、线程栈往往就能定位一半。Agent 不一样，它的问题常常发生在“哪一步选错了工具、哪一步上下文漂移了、哪一步 token 爆了”。

中文精读：

这份 Spring AI 文档最有价值的，不只是 Java 细节，而是它把 AI 系统可观测性拆成了非常工程化的对象。

文档里可以先抓住几个重点：

1. 工具调用是可观测对象
- tool calling 本身就应该被记录和统计

2. token usage 是核心指标
- 输入 token、输出 token、总 token 都应该可观测

3. 向量库操作也应该被观察
- add、delete、query 都有时延和次数

4. active 和 completed 指标不一样
- 一个看当前并发负载
- 一个看已完成请求的统计结果

这说明一件事：

Agent 可观测性不是“把最终答案打个日志”就够了，而是至少要能看见：

1. 模型调用次数
2. tool 调用次数和成功率
3. token 消耗
4. 检索耗时
5. 整个请求的时延
6. 失败发生在哪一步

对你后面的项目来说，最重要的是建立 step 级思维：

1. 用户输入进来了
2. 模型做了什么判断
3. 调了哪个工具
4. 工具返回了什么
5. 模型是否继续下一步
6. 最终为什么结束

你只需要记住的 3 个点：

1. Agent 的可观测性必须下沉到 step 级，而不只是接口级。
2. token、tool、vector store、延迟都是一等指标。
3. 没有轨迹和观测，Agent 系统几乎无法稳定迭代。

和 Agent 工程的关系：

你未来面试时如果能说清“我会看哪些指标、为什么这些指标重要”，会一下子从 Demo 视角切到生产视角。

今晚可动手：

给你的最小 Agent 服务设计一份日志字段草稿，至少包含：

- request_id
- task_id
- tool_name
- latency_ms
- token_in
- token_out
- status
