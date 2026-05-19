# title

2026-05-09 为什么 Agent 跑通以后，别只盯着 `finalOutput`，还要分清 `result surface` 和 `RunContext`

## original source

- 标题：Results | OpenAI Agents SDK
- 链接：https://openai.github.io/openai-agents-js/guides/results/
- 来源类型：官方文档
- 访问时间：2026-05-09

- 标题：Context Management | OpenAI Agents SDK
- 链接：https://openai.github.io/openai-agents-js/guides/context/
- 来源类型：官方文档
- 访问时间：2026-05-09

## why read today

昨天你已经补了 `handoff` 和 `session`，已经开始进入真正的 Agent 运行时视角。接下来最容易踩的坑，是把一次 `run()` 的结果理解得过于简单，脑子里只剩一句“模型最后回了什么”。这会直接导致后面三类工程问题变得很难处理：

1. 你不知道该把哪一层结果交给前端、哪一层结果留给调试和恢复流程。
2. 你会把本地运行依赖、用户上下文、工具状态，错误地塞进 prompt 或聊天历史里。
3. 一旦多 Agent、审批中断、工具调用、人工恢复这些场景出现，你的代码会迅速退化成“到处拼历史、到处传对象、到处打日志”。

对你这种做过 10 年 Java / Spring / IoT 集成 / ToB 交付的人来说，这一篇特别重要，因为它对应的其实是你熟悉的三个后端问题：

1. Controller 返回给调用方的 DTO，不等于系统内部所有运行态对象。
2. ThreadLocal / Session / Request Context / 依赖注入对象，各自有边界，不能混着用。
3. 一个可恢复、可观测的运行系统，必须分清“给业务看的结果”和“给运行时治理看的状态”。

## original-text translation

`Results` 这篇文档先把 Agent 运行后的结果面拆得非常明确。它说，当你执行一次 agent run 之后，拿到的不是只有一个“最终答案”，而是一组面向不同用途的结果字段。最常见的选择包括：

- 如果你只是要展示给用户看，用 `finalOutput`。
- 如果你要把完整本地转录继续带到下一轮，用 `history`。
- 如果你只想看这一次新生成的模型形态输出，用 `output`。
- 如果你要保留 agent、tool、handoff、approval 元数据，用 `newItems`。
- 如果你要处理待审批项和可恢复快照，用 `interruptions` 和 `state`。
- 如果你要拿应用上下文、审批状态、usage、嵌套 agent-tool 输入，用 `runContext`。
- 如果你要排查原始模型返回或 guardrail 诊断，用 `rawResponses` 和 guardrail result 数组。

它接着明确说明，`interruptions` 表示当前 run 因工具审批而暂停，`state` 是这个暂停态背后的可序列化快照。你可以先处理一部分审批，再把同一个 `state` 传回 `run()` 继续恢复，而不是重新拼接一轮新对话。

文档还强调了一点很容易被忽视的事实：`lastAgent` 或 `activeAgent` 才通常是下一轮应该继续接手的 agent，尤其当一次运行里发生了 handoff 时，最后完成工作的 agent 不一定还是最开始那个入口 agent。

最后，`Results` 把调试层面的几个字段也单独拉了出来：`runContext` 里不仅有你的应用 context，也有 SDK 管理的审批、usage 和嵌套 tool 输入；`rawResponses` 里保留了多步运行过程中可能出现的多次模型响应；usage 也不是散落在别处，而是聚合在 `result.state.usage` 和 `result.runContext.usage` 上。

`Context Management` 这篇文档则把“context”这个常被混用的词拆开了。它说，至少要区分两类 context：

1. 你的代码在本地运行时可访问的 context，比如工具依赖、用户 ID、logger、回调需要的状态。
2. 模型在生成答案时能看到的 Agent / LLM context。

文档明确说，本地 context 是通过 `RunContext<T>` 表达的。你创建自己的状态对象，传给 `Runner.run()`，然后工具、handoff 回调、生命周期 hook 都会拿到一个 `RunContext` 包装器来读写它。这个对象默认不会发送给 LLM，它纯粹是本地的，你可以安全地把依赖、用户标识、helper function 放进去，而不是把它们暴露给模型。

它还补了一条很有工程味道的说明：在同一个 run 里，即使有派生上下文或嵌套 `agent.asTool()` 调用，它们默认共享同一份底层 app context、approval 和 usage 跟踪；它们可能有不同的 `toolInput`，但不会自动隔离成彼此独立的应用状态副本。

## Chinese deep summary

如果把今天这两篇压成一句工程判断，那就是：Agent runtime 不是“输入一段话，返回一段话”，而是“运行出一组不同职责的结果面，并且在本地维护一份不应直接暴露给模型的运行上下文”。

先看 `Results`。

很多人一开始写 Agent，代码会自然长成下面这样：

1. `const result = await run(agent, input)`
2. `return result.finalOutput`

Demo 阶段这当然没错，但一旦系统稍微复杂一点，这种写法会让你在错误的抽象层上开发。因为 `finalOutput` 只解决了“最后展示什么”，却没有解决“过程里发生了什么”“如果中断了怎么继续”“下一轮谁接着处理”“这轮到底花了多少 token”“为什么 guardrail 把它拦住了”。

OpenAI 这篇 `Results` 文档真正重要的地方，不是教你背字段名，而是逼你承认：一次 Agent 运行的产出天然是分层的。

`finalOutput` 更像对外响应 DTO。它适合给用户看，但不适合做恢复控制。
`history` 更像完整会话账本。它适合继续对话，但不适合直接作为业务日志摘要。
`newItems` 更像运行事件流。它保留了 agent、tool、handoff、approval 元数据，适合做审计、调试、离线分析。
`interruptions` 和 `state` 则是工作流控制面。它们不是为了展示，而是为了让 run 可以暂停、审批、恢复、重试。
`rawResponses` 和 guardrail 结果数组则是诊断面。你不会把它们直接暴露给普通用户，但线上排障时非常值钱。

这套分层，对你这种后端背景的人其实很友好。它很像你以前会区分：

1. 返回给前端的响应体
2. 保存在数据库里的流程状态
3. 分布式调用里的链路日志
4. 运维排障时才会打开的原始报文

如果这些层你以前不会混，现在做 Agent 也不该混。真正的坑在于，LLM 项目特别容易让人偷懒，把所有东西都塌缩成“最后回答那段文本”。一旦你这么做，后面人工审批、失败恢复、评估复盘都会变得非常痛苦。

再看 `state` 和 `interruptions`。

这部分对 ToB 交付尤其关键。因为真实流程里，Agent 不可能永远一口气跑完。它会遇到审批、外部系统等待、人工确认、工具拒绝、权限补充。这时如果你脑子里只有聊天历史，就会倾向于“重新发一轮请求，让模型自己想起来之前发生过什么”。这在工程上是很脆弱的。

文档给出的思路更像工作流引擎：暂停时拿 `interruptions` 看待处理项，恢复时拿同一个 `state` 继续执行。也就是说，恢复动作不是“再聊一次”，而是“继续同一轮运行”。这和你过去理解的工单审批流、补偿流、异步回调恢复，其实是同一种系统思维。

再看 `lastAgent` / `activeAgent`。

昨天你已经看过 handoff，今天这一点会把那个概念再落地一步。很多人多 Agent 跑完以后，下一轮还是把输入打回最外层入口 Agent，因为他们默认“入口永远是入口”。但文档提醒你，真正应该接下一轮的，往往是这次最后活跃的 agent。这个点非常像请求被路由到某个专门处理域之后，后续状态应该由该域服务继续持有，而不是机械退回总入口。

然后看 `RunContext`。

这是今天最值得你建立肌肉记忆的概念之一。因为很多刚从后端转到 Agent 的工程师，会犯两个方向相反但同样危险的错误：

1. 什么都往 prompt 里塞，让模型“知道更多”。
2. 什么都往全局变量里塞，让工具“随便都能取到”。

`RunContext` 提供的是一条中间道路：把真正属于运行时、本地依赖、业务宿主的数据，放在显式传递的本地 context 里，而不是塞给模型看，也不是散落在不可控的全局状态里。

这个边界一旦立住，你后面的设计会稳定很多：

用户 ID、租户 ID、logger、DAO、审批器、feature flag，这些都该是 local context。
给模型看的业务事实、任务说明、约束规则，才应该进 instructions 或输入消息。
如果某个东西只是工具执行时需要，但模型根本不应看到，那就不该放进 prompt。

这一点对你过去做 Java 企业系统的经验衔接非常顺。你不会把数据库连接串、用户内部权限对象、审计开关直接塞进 Controller 的返回值；同理，你也不该把这些东西“翻译成自然语言”喂给模型，只为了让工具在后面能用。

文档里还有一个非常关键但容易漏掉的点：同一轮 run 里的派生上下文默认共享同一份 app context、approval 和 usage 跟踪。这个意思不是“状态随便改都没关系”，而是提醒你要把 context 当成真实的共享运行对象来设计。也就是说：

1. 它适合承载跨工具共享的运行依赖和有限状态。
2. 它不适合承载没有边界、谁都能乱改的大杂烩对象。
3. 你需要像设计请求级上下文那样设计它，而不是把它当成临时方便的黑箱缓存。

把 `Results` 和 `RunContext` 合起来看，今天最重要的升级是：你会开始把 Agent 应用理解成一个“宿主应用 + 运行时控制面 + 模型执行面”的组合系统，而不是“多写一点提示词的聊天程序”。

## 3 key takeaways

1. `finalOutput` 只是结果面中的展示层，不是完整运行结果；恢复、审批、调试、链路分析分别要看 `state`、`interruptions`、`newItems`、`rawResponses`、guardrail results。
2. `RunContext` 是本地运行时上下文，不默认暴露给 LLM；凡是工具依赖、用户标识、审批器、logger、usage 这类宿主侧数据，应优先放在这里，而不是硬塞进 prompt。
3. 多 Agent 和人工审批场景下，真正稳定的系统不是“重新聊一轮”，而是围绕 `lastAgent/activeAgent`、`state` 和共享 context 去继续同一轮运行。

## relation to Agent engineering

这篇内容和你后面做 Agent 工程的关系非常直接。

第一层，是宿主应用的接口分层。以后你不管是用 Python 写 Agent，还是最终用 Spring Boot 做宿主层，都应该尽早确定哪些字段是给用户界面的，哪些是给运行时恢复的，哪些是给观测和评估的。不要让 `finalOutput` 一层吞掉全部责任。

第二层，是工具执行边界。你过去做系统集成时，最怕“上下游都能改同一份隐式状态”。现在 `RunContext` 给了你一个显式边界：要共享，就在这里定义清楚；不该给模型看，就别塞进消息；不该跨 run 共享，就别偷偷做成全局单例。

第三层，是可恢复工作流思维。后面你无论接 MCP、审批流、外部 API、长任务执行，都会遇到暂停和恢复。今天这篇会帮你把“聊天机器人状态管理”升级成“运行状态管理”，这正是 Agent 工程和普通聊天封装之间的分水岭。

第四层，是你补 Python 和 Eval 的方式也会因此更稳。因为你后面做测试时，不会只断言最终文本，而会开始关心：

1. 是否产生了不该有的 handoff。
2. 是否留下了待审批 interruption。
3. usage 是否异常升高。
4. 工具调用输入是否来自正确的 local context。

这会让你的学习路径更像真正的工程落地，而不是只停留在 API 调用层。

## a small action for tonight

今晚花 30 分钟，给你未来最想做的那个 Agent 先画一张“结果面和上下文边界草图”，只写一页：

1. 先选一个场景，比如“代码库改造助手”或“客户环境排障助手”。
2. 写出它一次 `run()` 结束后，哪些内容给前端展示，只能来自 `finalOutput`。
3. 写出哪些内容必须保存在恢复层，比如 `state`、审批项、最后活跃 agent。
4. 写出哪些内容属于诊断层，比如 `newItems`、`rawResponses`、usage。
5. 再单独列一栏 `RunContext`，把用户 ID、租户、logger、工具依赖、审批器这些本地对象放进去。
6. 最后检查一句：这里面有没有什么内容其实不该让模型看到，却被你下意识写进 prompt 了。

## 原文关键段落翻译（人工翻译，放在文末）

1. 结果对象不是只提供一个最终答案；不同字段分别服务于展示、续跑、审批恢复、嵌套 agent-tool 元数据、原始模型响应和 guardrail 诊断等不同用途。
2. 如果你需要给下一轮传完整本地转录，用 `history`；如果你只要给用户展示最终回答，用 `finalOutput`；如果你要待审批项和可恢复快照，用 `interruptions` 和 `state`。
3. 当工具需要审批时，run 会暂停，待处理项会出现在 `interruptions` 中；处理完一部分或全部审批后，可以把同一个 `state` 重新传回 `run()` 继续恢复，而不必重新拼整段对话。
4. `runContext` 是结果上对运行上下文的公开视图，其中既有你的应用 context，也带有 SDK 管理的审批、usage 和嵌套 `toolInput` 等运行时元数据。
5. Context 需要至少区分两类：本地代码可访问的 context，以及模型生成回答时能看到的 context。
6. 本地 context 通过 `RunContext<T>` 表达：你把自己的状态对象传给 `Runner.run()`，工具、回调和生命周期 hook 都会收到一个包装器去读取或修改它。
7. 这个本地 context 默认不会发送给 LLM，它纯粹留在宿主应用侧，因此适合承载依赖、用户标识、helper function 和运行时状态。
8. 同一个 run 内部的派生上下文默认共享同一份底层 app context、approval 和 usage 跟踪；嵌套 `agent.asTool()` 可能有不同的 `toolInput`，但不会自动得到彼此隔离的应用状态副本。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.github.io/openai-agents-js/guides/results/
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.github.io/openai-agents-js/guides/context/
