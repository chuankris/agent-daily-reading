# title

2026-05-30 为什么现在要补 OpenTelemetry 的 GenAI / Agent 语义约定

## original source

- 标题：`Semantic conventions for generative client AI spans`
- 链接：https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/
- 来源类型：OpenTelemetry 官方规范
- 访问时间：2026-05-30

- 标题：`Semantic Conventions for GenAI agent and framework spans`
- 链接：https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/
- 来源类型：OpenTelemetry 官方规范
- 访问时间：2026-05-30

## why read today

你这轮补课到现在，已经把 Python 工具封装、MCP 协议层、Trace Context 这些“能把系统搭起来”的基础补了不少。接下来最容易出现的新短板，不是功能写不出来，而是：

1. 系统跑起来了，但你看不清一次 Agent 任务里到底慢在模型、工具、检索还是工作流编排。
2. 日志一堆，trace 也有，但不同团队、不同语言、不同 SDK 打出来的字段完全对不上。
3. 做了评估却很难复盘失败原因，因为“模型输出差”“工具参数错”“会话上下文丢了”都混在一起。

今天这两份规范，解决的正是这个阶段的问题。

它们不是在教你“怎么接一个观测平台”，而是在定义：**Agent 工程里哪些运行时事实，应该被记录成一套跨语言、跨框架、可对齐的公共语义。**

这对你尤其重要，因为你不是纯研究型转型路线，而是典型的工程交付型背景：10 年 Java / Spring / IoT 集成 / ToB 项目经验，意味着你真正的优势不在“最早会调模型”，而在“把复杂系统做成能排障、能协作、能交付的东西”。如果没有语义约定，Agent trace 很容易退化成新瓶装旧酒的日志拼盘；而一旦语义约定立住，你原来熟悉的调用链、关联 ID、错误分层、审计边界这些能力，就都能迁移过来。

更具体地说，这篇值得放在今天读，是因为它刚好接在你 2026-05-28 读过的 `W3C Trace Context` 后面。`Trace Context` 解决“链路怎么串起来”，而 OpenTelemetry 的 GenAI / Agent semantic conventions 解决“串起来以后，每一段到底该记什么”。前者是 trace ID 级别的通路，后者是 Agent 运行时语义级别的字典。两者合起来，才是能落到生产的观测基础。

## original-text translation

OpenTelemetry 在 GenAI 规范里并没有把重点放在“某家模型怎么记”，而是先定义一组更通用的 span 语义。对于生成式 AI 客户端调用，它把推理、embedding、retrieval、tool execution 这些典型操作抽成统一的 span 类型，并要求用一套共同字段去描述，比如操作名、提供方、模型名、错误类型、输入输出 token、工具名等。

规范特别指出，很多 tool 调用其实不是由自动埋点库天然捕获的，而是直接发生在应用代码里。因此，应用开发者应该主动遵循这套 tool 语义约定，并为自动埋点覆盖不到的 tool call 手工补充埋点。对 tool span 来说，核心字段至少包括工具名，最好还能带上 tool call id、工具描述、工具类型；参数和结果虽然也支持记录，但因为可能包含敏感信息，所以属于 opt-in，而不是默认强制。

在记录内容方面，规范态度很克制。它明确说，system instructions、用户消息、模型输出通常既大又敏感，不应该默认采集。更稳妥的做法是把“默认不记内容”作为基线；如果业务确实需要，可选择把内容写进 span 属性；如果到了生产环境，更推荐把原文放到外部安全存储里，trace 里只保留引用。

另一份 Agent / framework spans 规范是在前一份基础上再往上加一层。它把 agent 相关操作继续细分为：创建 agent、远程调用 agent、进程内调用 agent、调用 workflow 等几类 span，并约定这些 span 应该带哪些 agent 维度字段，例如 agent id、agent name、agent version、conversation id、data source id、output type、model、temperature 等。

这份规范还强调，`gen_ai.conversation.id` 用来标识会话或线程，并把消息关联回同一轮对话上下文；`gen_ai.data_source.id` 用来标识 RAG 或 agent 所依赖的数据源，但这个标识应尽量贴近 GenAI 系统实际使用的 ID，而不是简单复用底层数据库自己的名字。

另外一个很关键的点是状态成熟度。当前 GenAI / Agent semantic conventions 仍处于 development 状态，规范要求已有埋点实现不要默认强行切换到最新版，而应通过 `OTEL_SEMCONV_STABILITY_OPT_IN` 之类的机制让使用者显式选择是否启用新的实验版本。这说明它已经很有参考价值，但在生产落地时要带着“版本治理”意识去用，而不是当作完全冻结的标准。

## Chinese deep summary

如果把今天这两份规范压成一句话，就是：

**Agent 观测不是“把日志接到平台里”，而是先把一次 Agent 运行拆成若干有公共语义的 span，再让不同语言、不同框架、不同团队都按同一种字典记账。**

第一层，你要把“可观测性”从日志思维升级到运行时语义思维。

很多团队做 Agent 观测时，第一反应是多打几行日志，或者在每次模型调用前后记录耗时和 token。这样不能说没用，但远远不够。因为 Agent 系统不是单一步骤，它天然包含多个不同性质的环节：

1. 模型推理
2. 工具执行
3. 检索/数据源访问
4. 工作流编排
5. 会话上下文延续

如果这些环节都只记成“某次请求的一个耗时数字”，你最后看到的只是模糊的慢，而不是结构化的慢。OpenTelemetry 这套规范的价值，在于它不让你停留在“请求成功/失败”这一层，而是逼你把 Agent 过程拆成可比较、可归因、可串联的操作类型。

第二层，tool call 是 Agent 工程里最该手工补埋点的地方。

这一点对你尤其关键。你现在正在补 Python 和 Agent runtime，但你的强项其实一直是系统接入和交付边界。真实项目里最麻烦的问题，往往不是模型本身，而是 tool 层：

1. 参数到底是谁拼错了
2. 超时发生在模型侧还是企业接口侧
3. 同一个工具失败，是鉴权问题、网络问题，还是业务无数据
4. 工具结果是否包含敏感字段，不该直接进日志

规范明说，很多工具调用是应用代码直接执行的，自动埋点不一定捕得到，所以开发者应主动按约定手工打点。这句话非常值得你记住，因为它几乎等于在告诉你：**Agent 项目里最有工程价值、最体现基本功的观测，不在模型黑盒里，而在你自己可控的工具边界。**

第三层，内容采集必须和安全、成本、权限边界一起设计。

这点也很贴合你未来做 ToB Agent 的场景。很多人一做观测，就想把 prompt、模型输出、工具参数、工具结果全部存下来，觉得这样最好排查。但规范的建议是反过来的：默认不记内容；如果要记，显式 opt-in；生产里更推荐“内容外置，trace 留引用”。

这是很成熟的工程判断。因为在企业场景里，真正贵的从来不只是存储费用，还包括：

1. 敏感数据泄露风险
2. 不同团队对同一份内容的访问权限差异
3. 运维观测系统与业务数据系统之间的合规边界
4. 长期回放与审计的保留策略

你过去做 IoT 平台和集成项目，应该对这类边界感很熟。Agent 系统只是把问题换了个壳，本质没变。

第四层，`conversation id`、`agent version`、`data source id` 这些字段，决定了 trace 是否真的能服务评估。

很多团队做 eval 时，只盯最终答案对不对。但如果没有这些运行时字段，你很难回答下面这些更有价值的问题：

1. 某次失败是不是只发生在某个 agent 版本上
2. 某批错误是不是集中在某个数据源
3. 同一会话里模型多轮行为为什么前后不一致
4. 某个 workflow 节点是不是把上下文传断了

也就是说，这套语义约定不仅是“监控字段规范”，它其实还是 eval 和故障复盘的共同底座。没有统一语义，评估样本就很难和真实执行链路对齐。

第五层，这套规范现在最值得你学习的，不是“马上照抄全量字段”，而是它背后的建模方法。

当前规范还是 development 状态，这意味着你不该把它当作 100% 冻结标准，然后一口气在项目里硬编码所有字段。更现实的做法是：

1. 先抓稳定价值高的字段，例如 operation、provider、model、tool name、error type、conversation id。
2. 对敏感内容类字段坚持默认关闭。
3. 给语义版本留切换开关，而不是把实验字段写死在平台查询里。
4. 在 Java/Python 两边都用同一套 naming 做最小统一。

这很像你以前做协议对接或设备标准接入时的思路：先抓最小公共集合，再逐步扩展，不跟着实验版本乱漂。

第六层，这篇内容对你当前阶段的真正价值，是把“观测”从配套设施，提升为 Agent 基础设施本身。

你接下来无论是做 MCP server、tool gateway、RAG agent，还是一个带审批和恢复能力的多步骤执行器，迟早都会遇到一个现实问题：不是“能不能跑”，而是“出了问题以后，能不能在 10 分钟内定位是哪一层出了问题”。这套语义约定，就是为那个问题服务的。

## 3 key takeaways

1. OpenTelemetry 的 GenAI / Agent semantic conventions 不是简单多记几个字段，而是在定义 Agent 运行时的公共语义字典，方便跨语言、跨框架、跨团队对齐 trace。
2. Tool 调用是 Agent 工程里最值得手工补埋点的层，因为自动埋点未必覆盖，而真实故障大多恰好发生在工具边界、数据边界和权限边界。
3. `conversation id`、`agent version`、`data source id`、`error type` 这些字段不只是监控信息，它们直接决定你后续做评估、回放、审计和定位问题时有没有抓手。

## relation to Agent engineering

这篇和 Agent engineering 的关系非常直接，而且和你的转型路线高度契合。

第一，它会影响你以后怎么设计 tool 层代码。你不应该只关心“工具函数能不能跑通”，而要从一开始就考虑：这个工具有没有统一名字、有没有 call id、失败会不会被归成低基数错误类型、参数和结果是否需要脱敏采集。

第二，它会影响你怎么拆 runtime。以后你做 agent host 或 workflow executor 时，最好把 `invoke_agent`、`invoke_workflow`、`execute_tool`、`retrieval` 这些操作边界先想清楚，再决定代码模块和 trace span 的边界，否则实现会越来越像一锅难以归因的大函数。

第三，它会直接提高你的 eval 质量。你现在要补的不只是“怎么跑评测集”，还要补“怎么让失败样本能回到真实调用链路”。没有语义化 trace，eval 只会告诉你结果差；有了统一语义，你才能进一步区分是模型问题、工具问题、数据问题还是编排问题。

第四，它也给你的 Java 背景一个非常自然的落点。JVM 侧很适合先把企业接口、审批链、设备接口、内部服务封成稳定工具层，并在这一层把 OTel 语义打扎实；上层 agent 编排再用 Python 或 TypeScript 试验速度更快。这不是妥协，而是充分利用你已有优势。

## a small action for tonight

今晚做一个 30 到 40 分钟的小动作，不求接平台，只求把观测语义建起来：

1. 选你仓库里一个最小 agent 或 tool 调用流程。
2. 手写一张 span 设计表，只列 4 类：`invoke_agent`、`execute_tool`、`retrieval`、`workflow step`。
3. 每类只先定 5 到 8 个字段，优先放：`gen_ai.operation.name`、`gen_ai.provider.name`、`gen_ai.request.model`、`gen_ai.tool.name`、`gen_ai.conversation.id`、`error.type`。
4. 明确哪些字段默认不开，例如 prompt、工具参数原文、工具结果原文。
5. 最后用一句你熟悉的话检查设计是否合格：`如果线上一个客户工单说“Agent 偶发慢且偶发答错”，我能不能靠这套 span 在 10 分钟内判断问题落在哪一层？`

## 原文关键段落翻译（人工翻译，放在文末）

1. 生成式 AI 客户端调用应被记录为一组有统一语义的 span，例如推理、embedding、retrieval 和工具执行，而不是只记成笼统的一次请求。
2. 工具往往是由应用代码直接执行的，因此开发者应遵循这套工具语义约定，并为自动埋点覆盖不到的工具调用手工补埋点。
3. `gen_ai.tool.call.arguments` 和 `gen_ai.tool.call.result` 可能包含敏感信息，所以应作为可选采集项，而不是默认记录。
4. system instructions、用户输入和模型输出通常既敏感又体积大，OpenTelemetry 不建议默认采集，而应按场景显式开启。
5. 在生产环境中，更推荐把原始内容存到外部安全存储中，trace 中只保留引用，这样更容易控制成本和权限。
6. Agent / framework spans 在通用 GenAI spans 之上继续补充 agent id、agent name、agent version、conversation id、data source id 等字段，用来描述一次 Agent 运行的上下文。
7. `gen_ai.conversation.id` 用于把同一会话内的消息关联起来；`gen_ai.data_source.id` 应贴近 GenAI 系统实际引用的数据源标识，而不是随意复用底层数据库名称。
8. 当前 GenAI / Agent semantic conventions 仍处于 development 状态，已有埋点实现不应默认强行切换到新版本，而应通过显式 opt-in 方式让使用者选择。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/
