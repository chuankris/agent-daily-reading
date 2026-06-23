# title

2026-06-24 为什么现在该补 OpenAI Agents SDK 的编排与会话状态：别把多 Agent 当“自动协作”，先把控制权和记忆边界设计清楚

## original source

- 标题：Agent orchestration
- 链接：https://openai.github.io/openai-agents-python/multi_agent/
- 来源类型：OpenAI Agents SDK 官方文档
- 访问时间：2026-06-24

- 标题：Sessions
- 链接：https://openai.github.io/openai-agents-python/sessions/
- 来源类型：OpenAI Agents SDK 官方文档
- 访问时间：2026-06-24

## why read today

你前一阶段已经把 MCP、JSON-RPC、trace、pytest、asyncio、Spring AI 桥接这些点补得比较扎实了。现在最值得补的，不是再多看一篇“Agent 很智能”的介绍，而是把真正决定系统稳定性的两个问题讲透：

- 多 Agent 到底谁来控流程，什么时候该让模型自己决定，什么时候该由代码硬控？
- 多轮状态到底放哪，谁负责把历史取回、保存、恢复，而不是每轮都手工拼上下文？

这正好命中你当前的短板组合。

- 你有 10 年 Java / Spring / IoT 集成 / ToB 交付经验，对“职责边界、状态持久化、流程编排、故障恢复”非常熟。
- 你现在补的是 Python Agent runtime，最容易踩坑的不是语法，而是把多 Agent 协作和会话状态当成“框架帮我自动搞定”。
- 你之前已经读过 handoff、session 的基础概念，但今天这两篇 Python SDK 官方文档，把它们推进到了更像真实系统设计的位置。

今天这组材料的价值，不在于概念新，而在于它把你后面做 Agent 工程时最容易混掉的几条线拆开了：

- `agent as tool` 和 `handoff` 不是一回事。
- LLM 决策编排和代码控制编排不是一回事。
- `session` 不是“聊天记录”，而是运行连续性接口。
- 本地 session 和记在 OpenAI 服务器侧的 continuation 机制不能乱叠。

如果这几条边界今天立住，后面你再看 LangGraph、OpenAI Agents SDK、Spring 宿主层接 Python Agent、甚至 Eval/Trace 的时候，脑子里就不会只剩“一个会调用工具的大模型”。

## original-text translation

`Agent orchestration` 这篇文档先把问题定义得很工程化。所谓编排，就是在你的应用里决定哪些 agent 会运行、按什么顺序运行、由谁决定下一步做什么。官方把编排分成两大类：一种是让 LLM 自己做决定，也就是利用模型去规划、推理和选择下一步动作；另一种是由代码来编排，也就是把流程顺序明确写死在宿主逻辑里。两种模式可以混用，但它们各自有清晰的取舍。

文档随后指出，在 Python SDK 里，最常见的两种模式是 `agents as tools` 和 `handoffs`。`agents as tools` 的意思是，一个 manager agent 持续掌握对话控制权，把专门 agent 当成工具去调用；适合需要一个 agent 统一给出最终答案、汇总多个专家输出、或者在一个中心位置统一施加 guardrails 的场景。`handoffs` 则不同，它是把控制权转交给下一个 agent，更适合需要让专门 agent 接管后续对话、或者不同 agent 独立和用户直接交互的场景。

这篇文档还专门提醒：并不是所有流程都该让 LLM 自己决定。对于可预测、固定、重复性高的工作流，更适合由代码明确控制执行顺序；而对开放式任务、需要灵活规划的场景，才更适合把决策空间交给模型。这一点很像你过去做系统集成时区分“固定审批流”和“人工判断流”。

`Sessions` 这篇文档则把状态边界讲得非常清楚。官方直接说，Agents SDK 提供了内建 session memory，可以在多次 agent run 之间自动维护对话历史，从而不需要你在每轮之间手工处理 `.to_input_list()`。也就是说，session 的价值不是多一个方便参数，而是把“历史取回”和“新历史落盘”收口进统一机制。

文档的 quick start 例子很直接：给 `Runner.run()` 传入同一个 `SQLiteSession("conversation_123")`，第一轮问金门大桥在哪座城市，第二轮继续问“它在哪个州”，agent 就会自动记住上一轮上下文。官方还补了一条非常关键的边界：如果一次 run 因审批而暂停，恢复时应该继续使用同一个 session 实例，或者至少使用指向同一底层存储的 session，这样恢复后的 run 才会接上原来的对话历史。

更重要的是，官方明确写出 session 启用后的核心行为：每次 run 前，runner 会先把该 session 的历史取回来并拼到输入前面；每次 run 后，新的用户输入、助手输出、工具调用等条目都会被自动存入 session；之后再次用同一个 session 运行，就会带上完整上下文。这个定义说明 session 本质上是运行时记忆协议，而不只是“把消息存在某个列表里”。

文档还特别强调一个容易误用的点：如果你已经使用 `conversation_id`、`previous_response_id` 或 `auto_previous_response_id` 这类 OpenAI 服务端 continuation 机制，就不要在同一次 run 上再叠加 SDK session。也就是说，状态管理必须选一套主机制，而不是多套机制并存后再赌它们能自然一致。

最后，`Sessions` 文档给了多种实现选型：`SQLiteSession` 适合本地开发和简单应用，`RedisSession` 适合多 worker / 多服务共享低延迟状态，`SQLAlchemySession` 适合已经有生产数据库体系的应用。对你来说，这一段特别值钱，因为它把 session 从“SDK 小功能”抬升成了真正的后端架构决策。

## Chinese deep summary

如果把今天这两篇官方文档压成一句工程判断，就是：

**Agent 系统里最容易被误判的，不是模型能力，而是流程控制权和记忆边界。**

第一层，先把“多 Agent 协作”从想象里拽回工程里。

很多人一提多 Agent，就自动脑补成“几个智能体自己商量着把事做完”。这个理解太松，会让后面的架构和排障都变糊。官方文档其实在做一件更务实的事：它要求你先回答，当前这条链路到底是谁在控制。

- 是入口 agent 一直负责，只把某些子任务分派给专家 agent？
- 还是一旦识别出某个领域，就正式把后续对话转交给另一个 agent？
- 又或者，这条流程压根不该交给模型自由发挥，而是应该由代码一步一步写死？

这三个问题一旦混在一起，系统很快就会失控。因为你既不知道最终答案该由谁兜底，也不知道安全边界挂在哪，更不知道 trace 上看到的一次“转交”到底是正常动作还是设计错误。

第二层，`agent as tool` 和 `handoff` 的区别，本质上是“局部能力调用”和“控制权转移”的区别。

这是今天最值得你带走的判断。很多初学者会把所有多 Agent 协作都做成 handoff，看起来很“智能”，但工程上往往更脆。因为 handoff 一旦发生，意味着后续对话主导权变了，提示词边界变了，输出责任人也变了。

而 `agents as tools` 更像你熟悉的服务编排模式：总控服务仍然在，总控服务去调用几个专门能力，再由总控统一整合输出。这种模式特别适合：

- 最终答案必须统一口径；
- 多个专家输出需要汇总；
- guardrail、审计、审批、收尾动作必须集中控制。

反过来，handoff 更适合“从这一刻开始，就该由别的角色正式接管后续对话”的场景，比如客服路由、领域代理切换、人工审批前后责任主体变化。

对你这种做过大量系统集成和客户交付的人来说，这个区分并不陌生。你以前不会让一个流程节点既像 API gateway 一样总控，又像某个领域服务一样长期持有对话上下文。Agent 世界里也一样，区别只是在这里“切换接手人”的动作被框架显式命名成了 handoff。

第三层，LLM 编排和代码编排不是“谁更高级”，而是“谁更适合承担不确定性”。

官方没有鼓励你把一切都交给模型。相反，它明确告诉你：

- 开放式、探索式、需要动态规划的任务，更适合让 LLM 决定下一步。
- 固定、可预测、重复性强、合规要求高的流程，更适合由代码明确控制。

这一点很适合你当前阶段。因为你从 Java / Spring / ToB 交付转过来，很容易天然偏向“先把流程写死”；而 Agent 工程又会诱导新人走到另一个极端，觉得“既然模型会推理，那就都让它自己想”。今天这篇文档的价值，就是帮你建立中间那条专业判断：

**把不确定性交给模型，把确定性留给代码。**

这样做的结果不是保守，而是更容易验收、回放、调试和做 SLA。

第四层，`session` 不是聊天记录，而是运行连续性的统一接口。

这是很多 Agent Demo 和真实系统之间的分水岭。Demo 里多轮历史常常只是“把之前消息列表再传一遍”；但一旦进入产品化，你马上要面对的是：

- 对话跨多轮怎么自动取回？
- 工具调用、审批暂停后的恢复怎么接？
- 多 worker 场景下状态存哪里？
- 本地内存、SQLite、Redis、关系库分别适合什么阶段？

`Sessions` 文档把这些问题全提到了运行时层面。SDK 不是只帮你省掉一点样板代码，而是在告诉你：**多轮状态应该成为正式基础设施，而不是散落在业务代码里的字符串拼接。**

这和你过去在 Java 系统里做 session 管理、流程状态仓储、审批恢复其实是同一种工程意识。只是以前你保存的是用户会话、工单状态、流程实例；现在保存的是 Agent 的历史条目、工具运行痕迹和下一轮继续执行所需的上下文。

第五层，session 机制和服务端 continuation 机制不能混着管。

这一点特别像真实项目里的“状态主权”问题。OpenAI 文档明确说，session 不能和 `conversation_id`、`previous_response_id`、`auto_previous_response_id` 混用。原因很朴素：一段历史只能有一套主状态来源，否则你根本讲不清谁是真相。

这对你后面自己搭宿主层很重要。无论你将来是：

- 用 Python 服务自己管 Agent memory；
- 让 OpenAI 服务端帮你接续上一轮；
- 或者外面再包一层 Java / Spring 作为企业接入层；

你都必须先定清楚：历史的 authoritative source 到底是哪一层。否则故障恢复、回放分析、审计追责都会变得很难。

第六层，官方给出的 session 实现选型，正好能把你的旧经验迁到新栈里。

这部分很容易被读成“只是 SDK 列表”，但其实是架构建议：

- `SQLiteSession` 适合本地开发、单机原型、先把行为跑顺；
- `RedisSession` 适合多进程 / 多服务共享低延迟会话；
- `SQLAlchemySession` 适合已经有数据库治理和运维体系的生产应用。

这跟你以前选嵌入式库、本地缓存、MQ、关系库没有本质不同。Agent 工程不是推翻原有后端经验，而是在新的运行对象上重新做一遍“边界、存储、恢复、观测”的设计。

## 3 key takeaways

1. 多 Agent 协作里，`agents as tools` 代表“入口 agent 保持控制权”，`handoff` 代表“控制权正式转移”；二者不能混着理解。
2. 编排方式要按不确定性来选：开放任务更适合让 LLM 决策，固定流程更适合由代码显式编排，不要把一切都交给模型。
3. `session` 的核心价值不是保存聊天记录，而是统一处理历史取回、新条目持久化、审批后恢复和跨轮连续性；同时它不能与服务端 continuation 机制乱叠。

## relation to Agent engineering

这篇内容和 Agent engineering 的关系非常直接，因为它补的是“运行时骨架”而不是提示词技巧。

第一，它会影响你怎么拆 Agent 架构。以后你做代码助手、运维助手、企业知识问答助手时，应该先问“谁对最终输出负责，谁持有后续对话控制权”，而不是先堆更多 agent。

第二，它会影响你怎么做评测和排障。只有当你明确区分了 manager-agent 调专家、handoff 转控制权、session 持续历史，这些 trace 和 eval 指标才有解释力。否则同样一个失败样本，你会分不清到底是模型判断错、交接策略错、还是状态恢复错。

第三，它和你的背景非常匹配。你原来擅长的是流程编排、状态治理、交付期排障和系统边界设计；现在只是把这些能力从 Java/Spring/IoT 集成场景迁到了 Python Agent runtime。今天这篇官方文档正好是这个迁移的高价值接口。

## a small action for tonight

今晚做一个 45 分钟以内的小练习，不写大项目，只画边界：

1. 选一个你熟悉的 ToB 场景，比如“售后工单处理”或“设备告警排查”。
2. 画 3 个角色：`triage agent`、`specialist agent`、`manager agent`。
3. 对每一条协作关系标清楚：这里到底该是 `agent as tool`，还是 `handoff`，为什么。
4. 再补一页状态设计：如果流程中间要等人工审批，`session` 存哪，恢复时靠什么 `session_id` 找回。
5. 最后只写一句自己的工程结论：`多 Agent 的难点不是多写几个角色，而是先定控制权和状态主权。`

## 原文关键段落翻译（人工翻译，放在文末）

1. 编排指的是在应用里决定哪些 agent 会运行、按什么顺序运行，以及由谁来决定下一步发生什么；主要有让 LLM 决策和由代码编排两种方式，而且两者可以混用。
2. 在 Python SDK 里，常见的两种编排模式是 `agents as tools` 和 `handoffs`；前者适合由一个 manager agent 保持对话控制并整合专家结果，后者适合把后续对话正式交给另一个 agent。
3. Sessions 为 Agents SDK 提供了内建的会话记忆能力，可以在多次 agent run 之间自动维护历史，从而不需要你手工在轮次之间处理 `.to_input_list()`。
4. 启用 session 后，每次 run 之前 runner 会先取回历史并拼到输入前面；每次 run 之后，本轮产生的新条目会被自动保存到 session 中。
5. 如果一次 run 因审批而暂停，恢复时应继续使用同一个 session，或使用指向同一底层存储的 session 实例，以便恢复后的运行接上原有历史。
6. 如果你已经使用 `conversation_id`、`previous_response_id` 或 `auto_previous_response_id` 这类服务端 continuation 机制，就不要在同一次 run 上再叠加 SDK session。
7. `SQLiteSession` 适合本地开发和简单应用，`RedisSession` 适合多 worker / 多服务共享内存，`SQLAlchemySession` 适合已有数据库体系的生产应用。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.github.io/openai-agents-python/multi_agent/
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.github.io/openai-agents-python/sessions/
