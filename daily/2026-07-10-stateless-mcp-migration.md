# title

2026-07-10 为什么今天该读“Stateless MCP + Roots/Sampling/Logging 退场”：别再把 MCP 当长连接会话协议，要开始按可横向扩展的 HTTP 集成面来理解

## original source

- 标题：The 2026-07-28 MCP Specification Release Candidate
- 链接：https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/
- 来源类型：MCP 官方博客 / 规范发布说明
- 发布时间：2026-05-21
- 访问时间：2026-07-10

- 标题：SEP-2577: Deprecate Roots, Sampling, and Logging
- 链接：https://modelcontextprotocol.io/seps/2577-deprecate-roots-sampling-and-logging
- 来源类型：MCP 官方 SEP（标准增强提案）
- 创建时间：2026-04-14
- 访问时间：2026-07-10

## why read today

你这条学习线前面已经补过 MCP 的 lifecycle、transport、resources、prompts、roots、sampling、authorization，也已经开始接触 tool contract、OAuth、tracing。现在如果还把 MCP 理解成“客户端和服务端先建一个会话，再围绕会话持续交互”，那后面一旦进入真实工程，就会在三个地方一起吃亏：

- 部署思路会停留在“要粘性会话、要共享 session store、要特殊网关处理”；
- 状态设计会继续依赖隐式 transport session，而不是显式业务句柄；
- 技术判断会落后于协议演进，还在补 `roots / sampling / logging`，却没意识到官方已经明确把它们推向退场通道。

今天这两篇材料之所以值得优先读，是因为它们不是“小修小补”，而是在重新定义 MCP 的工程心智。

第一篇官方发布说明讲的是：**到 2026-07-10 这一天，MCP 还没正式切到 `2026-07-28` 终版，但 release candidate 已经在 2026-05-21 公布，而且官方明确写了终版计划在 2026-07-28 发布。** 也就是说，这不是远期猜想，而是已经进入迁移准备窗口的现实变化。

第二篇 SEP 讲的是：**官方开始主动缩协议核心面。** `roots`、`sampling`、`logging` 不是马上坏掉，但已经被标成 deprecated，意思是老实现还能继续跑，新的系统设计却不该再把它们当未来主路径。

这对你特别重要。你过去 10 年做 Java / Spring / IoT 集成 / ToB 交付，最擅长的本来就不是“背几个新概念”，而是把系统边界、状态模型、网关路由、部署约束和兼容迁移讲清楚。今天这组材料正好把 MCP 从“会调几个工具”拉回到你熟悉的那种系统设计视角。

## original-text translation

MCP 官方在 2026-05-21 发布的 release candidate 里，把这次变化的主轴说得很明确：**下一版协议的 headline 不是多一个 feature，而是协议层变成 stateless。** 以前在 `2025-11-25` 版本里，客户端先发 `initialize`，服务端返回 `Mcp-Session-Id`，后续请求都要带着这个 session id，这等于把请求天然绑到了某个服务端实例上。现在在 `2026-07-28` 这条新线上，同一个 `tools/call` 变成单个自包含请求，请求头里直接带 `MCP-Protocol-Version`、`Mcp-Method`、`Mcp-Name`，客户端信息和能力则放进 `_meta`，任何一个 server instance 都可以独立处理。

文档紧接着把工程含义点破了：以前远程 MCP server 如果跑在生产环境，往往需要 sticky session、共享 session store，网关还得深度理解报文；而 stateless 之后，可以直接挂在普通 round-robin 负载均衡器后面，还能基于 `Mcp-Method` 头做路由，客户端也能按 `ttlMs` 缓存 `tools/list` 结果。对做后端和集成的人来说，这不是“协议实现细节”，而是部署复杂度和扩展性在下降。

更关键的一点是，官方没有把“无状态”理解成“应用不能有状态”。release candidate 反而强调：如果业务需要跨调用保存状态，不要再把状态偷偷塞在 transport session 里，而是像普通 HTTP API 一样，显式返回一个 handle，例如 `basket_id`、`browser_id`，再让模型在后续工具调用里把这个 handle 当普通参数传回来。换句话说，状态没有消失，而是从“协议隐式状态”变成“业务显式状态”。

同一篇发布说明还把 server-to-client 交互重新整理了一遍。以前很多人容易把持续连接想成前提，现在官方要求 server 发起的中途请求只能发生在处理某个客户端请求期间，并通过 Multi Round-Trip Requests 返回 `inputRequired` 结果，再让客户端带着 `inputResponses` 和 `requestState` 重试原始调用。它的设计目标很明显：即使需要中途追问或确认，也要把整件事保持成可重试、可路由、可审计的无状态请求序列。

SEP-2577 则是在另一条线上做减法。文档先讲原则：MCP core 应该尽量 minimal and focused，那些采用率低、和现有替代方案重叠、却给所有 client/server 带来实现负担的能力，应该退出核心协议。于是官方把三类功能标记为 deprecated：

- `roots`：因为语义偏“信息提示”而不是强约束，采用率也低，很多场景用工具参数、resource URI、server 配置或环境变量表达得更清楚；
- `sampling`：概念上很强，但正确实现需要人类审批、模型选择、安全治理和 tool loop 支持，复杂度太高，而 server 直接接 LLM provider API 往往更直接；
- `logging`：协议内日志通道和成熟现成的 `stderr`、OpenTelemetry 体系重叠，继续放在 core 里不划算。

SEP 也刻意说明：deprecated 不等于立刻断供。线上的 wire-level 行为暂时不变，类型和 capability negotiation 都不删，目的是给生态发信号，让大家停止把这些能力当长期主路径，并预留迁移窗口。也就是说，老系统先别恐慌，但新系统别再重押。

把这两篇放在一起看，MCP 正在发生的事情就很清楚了：**一边把协议核心收紧成更像标准化 HTTP/JSON-RPC 集成面，一边把低采用、高复杂、语义不够硬的能力从“默认核心功能”往外挪。**

## Chinese deep summary

今天最该建立的判断，不是“我知道 MCP 要变 stateless 了”，而是下面这句话：

**MCP 正在从“带协议会话的智能工具连接层”，收敛成“可路由、可缓存、可追踪、可横向扩展的标准集成协议”。**

这句话对你这种背景的人很关键，因为它会直接改变你以后怎么画架构图、怎么拆状态、怎么设边界、怎么评估一个 MCP 方案是不是适合生产。

第一层，协议会话正在退场，显式业务状态要上位。

以前很多人第一次看 MCP，会自然把它往 WebSocket、长连接、IDE 内部双向通道那类东西上想：先握手、再持有 session、后面慢慢聊。这种心智在 demo 阶段问题不大，因为 demo 通常单实例、单用户、链路短、状态少。

但到了生产环境，这套心智会立刻带来一串工程代价：

- 负载均衡要做粘性路由；
- 服务端实例之间要共享 session store；
- 网关如果想做更细路由，往往得看 body；
- 一旦某个实例挂了，会话续命和恢复会变复杂；
- trace、cache、幂等重试也更难做干净。

release candidate 本质上是在把这些代价从协议层拿掉。`initialize` 没了，`Mcp-Session-Id` 没了，client info 和 capability 放进每次请求的 `_meta`，请求头里有清楚的 `Mcp-Method`、`Mcp-Name`。对一个做过企业 API、集成网关、设备平台的人来说，这其实很熟：**协议开始向“普通但干净的 HTTP 集成接口”靠拢。**

这会逼你换一种状态设计方式。以后真正有状态的东西，不该再依赖某个 transport session 偷偷保存，而应该像 REST / RPC 系统一样，显式生成 handle，再通过工具参数传递。这个变化非常适合你，因为你原来就更擅长“把状态对象化、标识化、审计化”，而不是把状态藏在连接背后。

第二层，无状态不等于功能变弱，反而更利于 Agent 编排。

很多人会误以为“session 去掉了，是不是协议退步了”。恰好相反。官方给出的显式 handle 模式，实际上更利于模型理解和组合状态。因为 `basket_id`、`browser_id` 这种东西一旦进入工具参数，它就成了模型可见、可推理、可转交、可追踪的对象，而不是隐藏在 transport 元数据里的黑箱。

这点对 Agent 工程尤其关键。Agent 不是简单的同步接口调用器，它经常要跨步骤传上下文、跨工具传对象、跨 handoff 传工作状态。如果状态只躲在会话里，模型自己看不见，也无法自然决定下一步该把哪个对象交给哪个工具。显式 handle 让状态第一次真正进入工作流语义层。

你可以把它类比成以前做系统集成时的单据号、工单号、设备会话号、批处理任务号。成熟系统从来不是“靠连接记住一切”，而是“靠显式业务标识串起一切”。MCP 这次变化，本质上是在向这种成熟设计靠近。

第三层，server-to-client 的交互也在从“连接态”转向“请求态”。

前面你已经读过 elicitation、sampling 这类 client-side feature。那时容易形成一个印象：客户端像宿主、服务端像对话参与者，双方持续有一条比较活跃的通道。现在新的无状态路线把这件事重新定界了。

官方要求 server 中途向 client 要输入时，只能发生在当前客户端请求的处理过程中，并通过 `inputRequired` + `requestState` 的 Multi Round-Trip Requests 模式继续。这种设计的味道非常“企业系统”：

- 不允许凭空弹出一次用户确认；
- 每次追问都要能追溯到某个已发起请求；
- 中断后可以重试；
- 换一个实例也能接着处理；
- 更容易审计，也更容易做故障恢复。

这意味着你未来设计 MCP server 时，不能再假设“我和 client 有条长期会话，我想什么时候问就什么时候问”。更合理的心智是：**每一次追问、确认、补参，都是某次原始调用的显式续段。**

第四层，`roots / sampling / logging` 被 deprecated，提醒你别再沿着旧学习路径继续加码。

这点非常容易被误读成“之前学错了”。不是。你之前补这些内容仍然有价值，因为你需要理解旧规范和现有生态。但从 2026-07-10 这个时间点往后看，学习重点要调了。

`roots` 的问题在于语义过软。它更像“建议 server 看哪些目录”，而不是强约束边界。对做生产系统的人来说，软提示通常不如显式参数、资源 URI、配置项来得可靠。

`sampling` 的问题在于理想很美，但实现责任太重。你要做人类审批、模型选择、安全控制、tool loop 支持，最后发现很多 server 还不如直接对接模型 API。对你这种准备做 Agent 工程的人，这个判断很实用：**别因为一个能力概念上优雅，就忽视它的实现负债。**

`logging` 被退场则更好理解。协议里再造一套日志通道，不如直接走成熟基础设施。你前几天刚补过 tracing 和 OpenTelemetry，这正好串起来了：协议层不应该背太多“平台基础设施已经做得更好”的事情。

第五层，这次变化特别适合你把旧经验迁移过来。

如果你只是纯 prompt 背景，看到这些变化，容易把它们当“规范版本更新”。但你不是。你长期做的是 Java / Spring / IoT / 集成 / ToB 交付，所以你更该把它翻译成下面这些熟悉问题：

- 有没有 sticky session 依赖？
- 能不能挂标准 LB？
- 状态是隐式连接态，还是显式业务态？
- 网关能不能基于头部做路由和限流？
- list 结果能不能缓存？缓存范围和 TTL 怎么定义？
- 日志和 tracing 是不是该回归统一观测基础设施？
- 旧能力是兼容保留，还是未来主路径？

一旦你用这套问题去看 MCP，就不容易再停留在“又多了几个 methods / capabilities”的表层。你会更快进入真正的工程判断：这个协议现在更适合被接进标准平台能力，而不是作为一个特殊玩具通道单独养着。

## 3 key takeaways

1. `2026-07-28` 版 MCP 的核心变化不是新增功能，而是协议层 stateless；`initialize` 和 `Mcp-Session-Id` 退出后，部署、路由、扩展和重试模型都会变。
2. 无状态协议不等于无状态应用；真正该保留的业务状态要通过显式 handle 和工具参数传递，而不是继续藏在 transport session 里。
3. `roots / sampling / logging` 已经被官方标记为 deprecated；老实现仍可兼容，但新系统不该再把它们当长期主路径来设计。

## relation to Agent engineering

这篇内容和 Agent engineering 的关系非常直接，因为它决定的是你以后把 Agent runtime 接进企业系统时，底座是“能演示”还是“能扩展”。

第一，它影响你的 MCP 架构选型。  
如果你后面要做企业内 Agent、知识库工具层、设备运维 Agent 或 Java/Python 混合工具总线，stateless MCP 会让远程 server 更像普通服务，而不是特殊长会话中间件。你在网关、LB、缓存、观测、弹性伸缩上的设计空间会明显更大。

第二，它影响你怎么建 tool state。  
以后不管是 Python Agent 还是 Java/Spring 包出来的 MCP server，都更应该返回显式业务句柄，例如 `task_id`、`ticket_id`、`device_session_id`、`browser_id`，让模型和工作流显式携带它们，而不是依赖连接状态偷偷续上下文。

第三，它影响你怎么安排后续学习顺序。  
从今天开始，MCP 学习重点更应该往这些方向偏：

- stateless transport 和请求边界；
- 显式状态句柄设计；
- tool schema 与 JSON Schema 2020-12；
- OAuth / OIDC 授权硬化；
- OpenTelemetry 级别的 tracing 与 observability；
- 兼容旧版 client/server 的迁移策略。

这比继续深挖 `roots / sampling / logging` 更值，因为它更接近未来一年里真正会出现在工程现场的问题。

## a small action for tonight

今晚做一个 30 分钟的小练习，不写代码，只做“会话态改造”：

1. 选一个你熟悉的 Agent 工具场景，比如“查设备后下发诊断建议”或“读取工单后补充处理意见”。
2. 假设它原来依赖隐式 session，写出 3 个原本藏在 session 里的状态。
3. 把这 3 个状态改写成显式 handle 或显式参数，例如 `device_context_id`、`work_order_id`、`request_state`。
4. 再补 4 行部署判断：是否还需要 sticky session、LB 能否 round-robin、哪些 list 结果可缓存、trace 应该在哪一层串起来。
5. 最后写一句复盘：`如果明天要把这个 MCP server 放进生产，我最先想删掉的是哪种隐式状态依赖？`

## 原文关键段落翻译（人工翻译，放在文末）

1. MCP 官方在 2026-05-21 的 release candidate 中明确表示：下一版规范的发布候选已经可用，终版计划在 2026-07-28 发布，而且这次包含 breaking changes。
2. 发布说明指出：在新版本里，协议层变成无状态；`initialize`/`initialized` 握手被移除，`Mcp-Session-Id` 也被移除，因此任意一个 MCP 请求都可以落到任意一个 server 实例上处理，不再需要协议层的粘性路由和共享 session store。
3. 文档同时强调：无状态协议不代表应用必须无状态；如果需要跨调用保存状态，server 应该返回显式 handle，例如 `basket_id`、`browser_id`，再让模型在后续调用里把它作为普通参数传回。
4. 新的 Streamable HTTP transport 要求带 `Mcp-Method` 和 `Mcp-Name` 头，这样负载均衡器、网关和限流器可以基于操作类型路由，而不用深度解析 body；`tools/list` 等列表或资源读取结果还可以依据 `ttlMs` 和 `cacheScope` 做缓存。
5. SEP-2577 说明：`roots`、`sampling`、`logging` 从包含该提案的规范版本开始进入 deprecated 状态；在弃用期内，线上的 wire-level 行为不变，不删除类型，也不改变 capability negotiation，目的是给生态发出迁移信号，而不是立刻打断现有实现。
6. SEP 对原因解释得很直接：`roots` 采用率低且语义偏弱；`sampling` 正确实现需要审批、模型选择、安全控制和 tool loop 支持，复杂度高且采用率低；`logging` 与 `stderr` 和 OpenTelemetry 等成熟观测基础设施重叠。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://modelcontextprotocol.io/seps/2577-deprecate-roots-sampling-and-logging
