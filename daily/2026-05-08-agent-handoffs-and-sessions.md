# title

2026-05-08 为什么 Agent 工程里，`handoff` 和 `session` 比“多写几个 prompt”更重要

## original source

- 标题：Handoffs | OpenAI Agents SDK
- 链接：https://openai.github.io/openai-agents-js/guides/handoffs/
- 来源类型：官方文档
- 访问时间：2026-05-08

- 标题：Sessions | OpenAI Agents SDK
- 链接：https://openai.github.io/openai-agents-js/guides/sessions/
- 来源类型：官方文档
- 访问时间：2026-05-08

## why read today

前几天你已经连续补了 MCP 的能力暴露、授权边界和人机交互边界，今天该往上走一层，补“Agent 运行时怎么交接、怎么记忆”。这一步很关键，因为很多人做 Agent 时会先陷进去写 prompt、堆 tool、拼 workflow，但真正到了工程落地，最先失控的通常不是模型回答质量，而是这两个问题：

1. 什么时候应该把对话正式交给另一个 Agent，而不是让当前 Agent 一直硬撑。
2. 对话历史和运行状态到底由谁持有，怎么在多轮和中断恢复里保持一致。

对你这种做过 10 年 Java / Spring / IoT 集成 / ToB 交付的人来说，这和你过去熟悉的“请求分发”和“会话状态管理”其实是同一类工程问题，只是对象从 HTTP 接口和业务模块，换成了 Agent、tool 和模型上下文。

`handoffs` 解决的是“职责切换”，`sessions` 解决的是“记忆持久化”。把这两个概念补上，你看 Agent 系统就不会只盯着 prompt，而会开始用真正的平台视角去看运行边界。

## original-text translation

`handoffs` 这篇文档先把定义讲得很直接：handoff 允许一个 Agent 把对话中的一部分任务委托给另一个 Agent，这适合用于不同 Agent 各自负责特定领域的场景。它还特别强调，handoff 在模型看来是以 tool 的形式出现的。也就是说，模型不是凭空“换了个 Agent”，而是在当前运行里选择了一个类似 `transfer_to_refund_agent` 这样的交接工具。

文档接着区分了两种常见模式。如果你已经知道“该由专门 Agent 接管后续对话”，就应该使用 handoff；如果只是想让另一个 Agent 在后台完成一项局部任务，而最终仍由当前 Agent 对外负责，就更适合把那个 Agent 当作 tool 使用，而不是做 handoff。

在 handoff 的实现上，文档说每个 Agent 都可以配置 `handoffs`，既可以直接传另一个 Agent，也可以通过 `handoff()` 辅助函数做更细的定制。你可以给 handoff 配结构化输入 `inputType`，让模型在交接时一并附带一个小的结构化载荷；也可以用 `onHandoff` 在交接发生时记录日志、做埋点或者同步运行时状态。

还有一个特别工程化的点是 `inputFilter`。默认情况下，新的 Agent 会收到整个对话历史；如果你不想把所有历史都原样传过去，可以提供 `inputFilter` 改写传递内容。文档列出的 `HandoffInputData` 包括运行开始前的历史、handoff 前已经产生的条目、当前轮新产生的条目以及运行上下文。

文档还专门提醒了一个容易误解的边界：handoff 仍然发生在同一个 run 内。输入 guardrail 只作用于链路里的第一个 Agent，输出 guardrail 只作用于最终产出结果的那个 Agent。如果你需要对每次工具调用都做检查，应该使用 tool guardrail，而不是误以为 handoff 会自动把所有安全检查层层继承下去。

`sessions` 这篇文档讲的是另一层问题。它把 session 定义为 Agents SDK 的持久化记忆层：只要给 `Runner.run` 传入一个实现了 `Session` 接口的对象，SDK 就会自动在每次运行前取回已存历史、在每次运行后持久化新的用户输入和助手输出，并让这个 session 能继续用于之后的新一轮输入，或者用于从被中断的 `RunState` 恢复。

文档因此强调，使用 session 后，你不需要再手工调用 `toInputList()` 或自己拼接多轮历史。SDK 已经提供了 `MemorySession` 和 `OpenAIConversationsSession` 两类实现，同时也允许你接入自己的存储后端。它还补了一条很实用的提醒：如果你已经在用 `conversationId` 或 `previousResponseId` 这类 OpenAI 服务端托管状态，一般就不需要再为同一段历史额外叠一个 session。

最后，sessions 文档把“审批暂停再恢复”说得很明确。很多 human-in-the-loop 流程会在等待批准时暂停当前 run，而恢复时应该继续复用同一个 session，这样前后的历史和状态才能保持连贯，而不是每次审批完都像重新开了一个新会话。

## Chinese deep summary

如果把今天这两篇压成一句话，那就是：Agent 工程里的复杂度，不只是“模型会不会调工具”，更在于“职责什么时候换人”和“历史由谁记住”。

先看 `handoff`。

很多刚接触 Agent 的人，会把一个复杂任务都塞给一个总控 Agent，然后在 prompt 里告诉它“你可以根据需要调用不同工具”。这种做法在 Demo 阶段没问题，但一旦任务域变多、对话轮次变长、职责边界变复杂，系统就会开始出现一种很典型的退化：主 Agent 什么都想管，提示词越来越长，工具描述越来越乱，最后既难调试，也难评估。

`handoff` 的价值，是把“谁负责后续对话”这件事显式建模出来。它不是普通的函数调用，不是让另一个组件在后台做个子任务再回来，而是真正把会话控制权交给下一个 Agent。这个区分对工程稳定性很重要。因为一旦你承认“现在该由 B 接手”，你就可以给 B 独立的提示词、独立的能力范围、独立的输出约束，而不是让 A 永远背着所有上下文和所有职责。

这点和你过去做系统集成其实很像。以前在 Java / Spring 项目里，你不会让一个 Controller 同时承担协议适配、流程编排、库存核算、审批状态流转和供应商接口补偿。你会拆服务，会定边界，会明确“这个请求从这里开始转给谁处理”。`handoff` 本质上就是 Agent 世界里的这类边界声明。

文档里最值得你注意的，是“handoff 在模型眼里是一个 tool”。这意味着交接并不是框架在外面偷偷做的黑箱魔法，而是模型决策的一部分。工程上这会带来两个直接后果。

第一，你必须像设计重要工具一样设计 handoff 描述。什么时候该转交，交给谁，应该附带什么结构化上下文，这些都直接影响模型是否会在正确的时机做正确的交接。

第二，你要把 handoff 当成一种可观测的运行事件，而不是“代码分层细节”。既然它是显式的运行项，你就应该记录它、评估它、分析它是否转交过早、过晚、还是转错了对象。这和后面做 eval、trace、故障排查是连起来的。

再看 `inputFilter`，这部分尤其像真实工程，而不只是 SDK 用法。默认把全部对话历史交给下一个 Agent，虽然方便，但很容易带来两个问题：一是上下文噪声越来越大，二是把不该传的中间工具细节也一并传过去。`inputFilter` 的存在提醒你，handoff 不是“全量甩锅”，而是“带着经过整理的交接包换人”。这和你以前做系统交接时会整理上下文、提炼工单摘要、去掉无关日志，本质是一个动作。

然后是 guardrail 边界。文档明确说，input guardrail 只对第一个 Agent 生效，output guardrail 只对最终产出 Agent 生效。这句话非常值得单独记，因为很多人会天然假设：“既然是一个 agent chain，安全校验肯定会自动覆盖整条链。”现实不是这样。如果你需要每个工具调用、每次子动作都经过检查，你必须单独把控制点布进去。这个思维很像你以前做网关和服务治理时，不会因为入口有鉴权，就默认内部每一步都天然安全。

再看 `session`。

很多 Agent Demo 看起来能跑，是因为它只演示单轮。真正变成产品后，问题就会立刻从“会不会回答”变成“前一轮说过什么、这轮暂停到哪、审批后怎么接着跑、下次打开还能不能接上”。这时候 session 就不再是一个“方便封装”，而是运行时的记忆边界。

`sessions` 文档最重要的价值，是把“历史拼接”从业务代码里拿出来，交给统一接口处理。你不再需要在每次请求里手工回捞消息、拼接上下文、保存输出，这相当于把多轮对话状态管理提升成了框架层能力。对你来说，这可以直接类比成过去 Web 应用里统一的 session 管理、状态仓储和事务边界，只不过这里保存的不再是购物车或登录态，而是 Agent 的对话和运行轨迹。

更关键的是，它把“持久对话”和“可恢复运行”连在了一起。文档里提到，session 不只是给下一轮对话用，也能用于从中断的 `RunState` 恢复。这一点在 Agent 场景下特别重要，因为很多流程天然会被中断：等人审批、等外部系统返回、等用户补充信息、等人工复核。没有可恢复的 session，你的 Agent 流程就会像每次都重新开工；有了它，你才能把一次任务看成真正跨时间延续的运行。

对于你当前阶段，我觉得最值得建立的工程直觉是这两个判断：

1. `handoff` 解决的是职责分工问题，不是单纯的代码复用问题。
2. `session` 解决的是运行连续性问题，不是单纯的聊天记录保存问题。

当你把这两个判断立住后，后面再看 LangGraph、OpenAI Agents SDK、Spring AI Agent、Eval 和 Trace，会更容易形成一张完整图：Agent 不是一个“会聊天的大函数”，而是一套带边界、带状态、可交接、可恢复、可观测的运行系统。

## 3 key takeaways

1. `handoff` 代表的是会话控制权的正式转移；如果只是想让别的 Agent 在后台完成一段局部工作，应该优先考虑“agent as tool”，不要把所有协作都做成 handoff。
2. `inputFilter` 和 guardrail 边界说明了 handoff 不是自动继承一切上下文与控制规则；交接内容和控制点都需要你显式设计。
3. `session` 的核心价值不是“记住聊天历史”，而是把多轮状态持久化、中断恢复和运行连续性纳入统一接口，避免业务代码自己拼上下文。

## relation to Agent engineering

这篇内容和你后面做 Agent 工程的关系非常直接。

第一层，是多 Agent 架构怎么拆。以后你做代码助手、运维助手、知识库问答助手时，不要一上来就做一个“万能总控 Agent”。你应该先问：哪些场景需要真正接管后续对话，哪些场景只是后台调用一个专门能力。这会直接决定你用 handoff 还是 tool，也会影响评估和排障难度。

第二层，是状态托管放在哪。你过往的 Java 背景很容易让你自然想到“把对话历史存在数据库里”，这个方向没错，但今天这篇提醒你，不应该只把它当成存表设计，而要把它看成 `Session` 接口背后的运行契约：何时加载、何时合并、何时恢复、是否和 OpenAI 服务端状态重复。

第三层，是你以后补 Spring AI / Java 桥接时会更稳。因为你会发现，哪怕最终某些 Agent 运行时在 Python 侧，宿主系统的治理思路仍然和你熟悉的后端平台工程一致：职责切分、状态持久化、审批恢复、埋点追踪、边界控制。变化的是语言和 SDK，不变的是工程抽象。

## a small action for tonight

今晚花 30 分钟，画一张你自己的 Agent 运行草图，只要一页：

1. 选一个你最可能先做的 Agent，比如“代码库改造助手”。
2. 先画一个入口 Agent，再画两个专职 Agent，例如“代码理解 Agent”和“变更执行 Agent”。
3. 对每条连线标清楚：这是 `tool call` 还是 `handoff`，为什么。
4. 再补一块 `session` 说明：历史存在哪、审批暂停后怎么恢复、哪些状态必须跨轮保留。
5. 最后检查一句：如果明天让你用 Spring Boot 当宿主层，你会把 handoff 事件和 session 存储放在哪个模块里。

## 原文关键段落翻译（人工翻译，放在文末）

1. Handoffs 允许一个 Agent 把对话中的一部分任务委托给另一个 Agent，这特别适合不同 Agent 分别专注于不同领域的场景。
2. 在模型看来，handoff 是以工具的形式出现的；如果交接目标叫 `Refund Agent`，那么对应的工具名会是类似 `transfer_to_refund_agent`。
3. 默认情况下，handoff 会把整个对话历史交给新的 Agent；如果你想改变传递内容，可以提供 `inputFilter` 来改写输入数据。
4. Handoff 仍然发生在同一个 run 内；输入 guardrail 只对链路里的第一个 Agent 生效，输出 guardrail 只对最终产出结果的 Agent 生效。
5. Session 为 Agents SDK 提供持久化记忆层；当提供了 session 后，runner 会自动取回已保存历史、保存本轮新输入输出，并把该 session 继续用于未来轮次或中断恢复。
6. 使用 session 后，就不需要再手工把历史转成输入列表或自己在轮次之间拼接对话。
7. 如果已经使用 `conversationId` 或 `previousResponseId` 这类服务端托管状态，通常不需要再为同一段历史额外叠加一个 session。
8. 在 human-in-the-loop 流程里，run 经常会为了等待审批而暂停；恢复时应继续使用同一个 session，保证前后历史连贯。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.github.io/openai-agents-js/guides/handoffs/
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.github.io/openai-agents-js/guides/sessions/
