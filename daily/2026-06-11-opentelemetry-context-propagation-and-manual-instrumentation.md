# title

2026-06-11 为什么现在该补 OpenTelemetry 的 context propagation 和 Python 手工埋点

## original source

- 标题：`Context propagation`
- 链接：https://opentelemetry.io/docs/concepts/context-propagation/
- 来源类型：OpenTelemetry 官方文档
- 访问时间：2026-06-11

- 标题：`Instrumentation`
- 链接：https://opentelemetry.io/docs/languages/python/instrumentation/
- 来源类型：OpenTelemetry Python 官方文档
- 访问时间：2026-06-11

## why read today

你这轮补课到现在，前面已经把 `asyncio`、队列收尾、MCP 基础协议、Trace Context、GenAI 语义约定这些“零件级知识”补了不少。今天最该往前推的一步，不是继续记一个新 API，而是把这些零件接成一条能落到真实 Agent 宿主里的观测链路。

原因很现实：你现在的短板已经不只是“知道概念”，而是“能不能把 Python 里的一次 agent 调用、一次 tool 执行、一次外部 HTTP 请求、一次失败重试，稳定地串成一条 trace”。这正是 context propagation 和 manual instrumentation 解决的问题。

这对你尤其重要，因为你不是纯 Python 出身。你有 10 年 Java / Spring / IoT 集成 / ToB 交付背景，天然更擅长看调用链、边界、失败归因、跨系统排障。问题是，到了 Python Agent 生态里，很多人会停留在“框架自动打点了就算完”，但真正能交付的系统，最后总会卡在你自己写的那层代码上：

1. tool 包装层是谁起的 span。
2. 并发子任务和主流程怎么保持上下文。
3. 外部服务 header 里的 trace 信息怎么注入和提取。
4. 失败时只看到异常，还是真能回到具体 span、具体 tool、具体会话。

如果今天不把这层补上，后面做 Agent runtime、MCP host、Eval runner、甚至 Java 服务外挂 Python Agent 时，很容易出现一种表面上“功能能跑”，但实际上“出了问题根本追不回去”的假稳定。

## original-text translation

OpenTelemetry 对 context propagation 的定义很直接：它让 traces、metrics、logs 这些不同信号即使分布在不同服务、不同进程、不同执行单元里，也还能被关联到同一条因果链上。发送方会把 trace ID、span ID 等上下文放进请求元数据里，接收方再把这些值提取出来，作为本地新 span 的父上下文，这样整条请求路径就能串起来。

文档同时强调，传播通常由 instrumentation 库自动处理；但如果没有现成支持，开发者就应该自己用 Propagators API 做手工注入和提取。换句话说，自动化只是“能省事时省事”，不是“你可以不理解传播机制”。

在安全上，OpenTelemetry 也说得很克制：面对不受信任的外部服务时，传入上下文不能无脑相信，传出上下文也不能无脑全发；尤其是 baggage 里不要放凭证、密钥、PII 这类敏感数据，因为它会跨服务传播。

Python instrumentation 文档则把“你该如何在自己代码里落地”讲得更具体。它先区分 app 和 library：给应用做埋点，要装 SDK 并初始化 `TracerProvider`；给库做埋点，只依赖 API，让是否真正输出 telemetry 的决定权留给宿主应用。

在 span 创建上，文档推荐最常见的方式是 `start_as_current_span()`，因为它会把新 span 设成当前 span，天然形成父子关系。另一方面，如果你要追踪并发或异步操作，也可以用 `start_span()` 创建一个不自动成为 current span 的 span，这说明 OpenTelemetry 默认建模假设并不是“所有工作都严格串行”。

文档后面又补上了几个关键动作：当前 span 可以追加 attributes、events、status、exceptions，也可以用 links 把两个有因果关系但非父子的工作关联起来。默认传播格式则是 `W3C Trace Context` 和 `W3C Baggage`，这和你前面补过的 Trace Context 正好接上。

把两篇文档连起来看，本质上就是一句话：**trace 不是平台替你“收日志”，而是你自己在代码边界上维护上下文、声明操作语义、补齐失败证据。**

## Chinese deep summary

今天这篇最值得你记住的核心，不是某个 import 路径，也不是某个 SDK 初始化片段，而是这个判断：

**Agent 工程里的 tracing，难点不在“有没有接 OpenTelemetry”，而在“你自己写的那些 Python 边界，是否真的在传递上下文、创建正确的 span，并把失败附着回同一条链路”。**

第一层，为什么 context propagation 比很多人想象的更重要？

因为 trace 真正解决的不是“多打一层日志”，而是“把跨边界的动作还原成一条因果链”。一次 Agent 执行在现实里往往会跨这些边界：

1. HTTP 入口到 Agent runtime。
2. Agent runtime 到模型调用层。
3. 模型决策到 tool execution。
4. tool execution 到外部服务或数据库。
5. 主任务到并发子任务、后台任务或回调。

只要其中任意一段没把上下文带过去，你最后看到的就不是一条 trace，而是一堆彼此无关的局部成功与局部失败。对做过企业集成的人来说，这就像你以前排查一条 IoT 设备上报链路时，只拿到了网关日志，却没有 request id、device id、上游调用 ID 一样，系统不是不能跑，但出事时定位速度会极慢。

第二层，为什么今天选 OpenTelemetry 官方文档，而不是再追某个新 Agent 框架？

因为这部分知识更稳定，而且迁移性更强。框架会换，SDK 会升级，Agent runtime 会不断变，但“上下文如何传播”“span 如何作为当前执行单元被继承”“异常如何挂回 span”“不信任边界如何处理传播数据”这些原则不会变。

你现在的阶段，不缺再看一个“十分钟集成 tracing 平台”的教程，缺的是把 tracing 当成宿主工程设计的一部分。OpenTelemetry 这里给的是标准工程视角，而不是平台营销视角。

第三层，Python 手工埋点为什么是你现在最该补的？

因为 Agent 系统里最值钱、也最容易出问题的那层，恰好就是自动埋点不一定覆盖到的部分。比如：

1. 你封装的 tool registry。
2. 你自己写的 retry / timeout / fallback。
3. 你自己管理的并发 fan-out。
4. 你针对 MCP server、浏览器工具、文件工具、企业接口做的适配层。

这些地方框架通常只能看到“函数被调用了”，看不到“为什么这里要单独开 span”“这个错误应该归类成 tool 参数错误还是下游接口错误”“这个并发任务和父任务是父子关系还是 link 关系”。这就是 manual instrumentation 的价值。

第四层，`start_as_current_span()` 和 `start_span()` 的差别，为什么对 Agent 很关键？

因为它背后其实是在逼你想清楚执行模型。`start_as_current_span()` 适合表示“这个动作就是当前主路径的一部分”，例如一次 tool invocation、一次 retrieval、一次 workflow step。它天然会把当前 span 作为父节点继承下来。

而 `start_span()` 不自动设为 current span，更适合那些并发、异步、旁路的工作。对 Agent 来说，这类场景非常常见：主流程拉起多个候选工具探测、异步记录评测样本、后台上传工件、并发抓多个网页。你不能把所有工作都粗暴塞成父子树，否则 trace 看起来完整，语义却是错的。

这和你过去做系统集成时的经验很像：有些调用是严格主链路，有些只是旁路通知，有些只是异步补偿。链路图如果不分清这些关系，排障时会误判责任边界。

第五层，attributes、events、status、exceptions、links 这些字段在 Agent 里该怎么理解？

最实用的理解方式是：

1. attributes：给当前操作贴“结构化标签”，例如 tool 名、会话 ID、数据源 ID、重试次数、tenant、目标系统。
2. events：记录 span 生命周期里的关键节点，像“开始探测工具”“收到外部响应”“进入 fallback”这种瞬时事件。
3. status：告诉后端这个 span 最终是否算错误结束。
4. record_exception：把异常对象本身挂回 span，方便你从 trace 直接回看失败。
5. links：当两个工作有因果关系但不是严格父子时，用 link 表达，而不是硬塞层级。

这套东西对你这种准备补 Eval 能力的人尤其重要。因为 eval 不是只看最终答案对不对，还要能把失败样本回挂到真实执行链路。如果 span 上没有这些语义，评测最后只会变成“这个样本错了”，却不知道错在模型、错在 tool、错在 retrieval，还是错在 workflow 编排。

第六层，传播安全为什么要现在就建立意识？

很多初学者一接 tracing，就默认“能传的都传”。但 OpenTelemetry 官方明确提醒，不可信外部服务的 incoming context 不能盲信，outgoing context 也不能乱发，baggage 更不要塞敏感信息。

这点对你未来做 ToB Agent 很关键。企业环境里，trace header 不是天然可信输入；baggage 也不是“顺手塞点用户信息”的便利通道。你过去做交付时应该很熟悉这个原则：跨系统边界要先判断信任模型，再决定哪些元数据可以透传。Agent tracing 只是把这个老问题换了个壳。

第七层，把今天内容压缩成一个更贴近你转型阶段的工程动作，就是：

先别急着追“观测平台接没接全”，先在你自己的 Python 宿主里把下面几件事做对：

1. 主入口是否创建了顶层 span。
2. tool / retrieval / workflow step 是否各自有独立 span。
3. 并发支路是父子关系还是 links，是否表达正确。
4. 异常是否同时设置 status 并 record_exception。
5. 跨 HTTP / MCP / 外部接口时，header 或 carrier 里的上下文是否真的注入和提取了。

这五件事一旦做对，你后面无论换 LangGraph、OpenAI Agents SDK、Spring AI、还是自建 runtime，观测骨架都不会散。

## 3 key takeaways

1. Context propagation 的本质，是让 trace 在服务、进程、线程、异步任务这些边界之间保持同一条因果链；少一段传播，就少一段真相。
2. Python 手工埋点最有价值的地方，不是替代自动埋点，而是补齐自动埋点覆盖不到的 Agent 边界：tool 包装层、并发分支、失败归因、外部系统适配层。
3. 对 Agent 评测和排障来说，span 不只是“计时块”，还必须带上 attributes、events、status、exceptions、links，才能把失败样本回挂到真实执行路径。

## relation to Agent engineering

这篇内容和 Agent engineering 的关系非常直接。

第一，它决定你以后写的 Agent 宿主是不是“可定位”的。很多 demo 里 tracing 看起来已经接上了，但一到真实项目，自定义工具、重试、超时、fallback、并发子任务一多，自动埋点覆盖立刻不够，最后还是要回到手工埋点和上下文传播。

第二，它会直接影响你补 Eval 的效率。你现在缺的不是“知道评测集是什么”，而是“失败样本能不能回到真实 trace”。没有 context propagation 和结构化 span，eval 很快就会变成一张分数表，而不是工程改进工具。

第三，它很适合作为你 Java 背景迁移到 Python Agent 的桥梁。你以前熟悉的是调用链、MDC、拦截器、跨服务关联 ID、审计边界；今天学的只是把这套工程直觉翻译到 OpenTelemetry 和 Python 运行时里。这个迁移非常值钱，因为它比单纯会调某个 Agent 框架更接近真实交付能力。

## a small action for tonight

今晚做一个 40 分钟的小实验，不追求接平台，只追求把上下文和 span 边界做明白：

1. 写一个最小 Python demo：HTTP 入口里创建顶层 span，内部依次调用一个“模型决策函数”和一个“tool 函数”。
2. 给“模型决策函数”和“tool 函数”各自加 `start_as_current_span()`，并给 tool span 打上 `tool.name`、`conversation.id`、`retry.count` 之类的 attributes。
3. 在 tool 函数里故意制造一次异常，同时设置 `StatusCode.ERROR` 并 `record_exception()`。
4. 再加一个并发子任务，用 `start_span()` 或 link 明确表达它和主链路的关系，不要默认全塞成父子。
5. 最后检查自己能不能回答一句工程问题：`如果线上某个会话里的 tool 偶发失败，我能不能从一个 trace 一路看到入口、决策、tool、异常和并发支路？`

## 原文关键段落翻译（人工翻译，放在文末）

1. 上下文传播让 traces、metrics、logs 等信号即使在不同位置产生，也还能彼此关联；在 tracing 里，它用来跨服务边界建立完整因果链。
2. 当服务 A 调服务 B 时，A 会把 trace ID 和 span ID 放进上下文；B 用这些值创建属于同一 trace 的新 span，并把 A 的 span 作为父节点。
3. 传播是把上下文在服务和进程之间移动的机制，通常由埋点库自动处理；如果没有现成支持，开发者可以自己使用 Propagators API 做手工传播。
4. 面向不可信外部服务时，传入上下文要谨慎接受，传出上下文也要谨慎发送；baggage 中不应放凭证、密钥或个人敏感信息。
5. 给 Python 应用做埋点时，需要使用 OpenTelemetry SDK 初始化 `TracerProvider`；给库做埋点时，通常只依赖 API，把是否真正输出 telemetry 的决定权留给宿主应用。
6. 创建 span 时，最常见的是把它作为 current span 启动；如果想追踪并发或异步操作，也可以创建一个 span 但不把它设为 current span。
7. attributes 用来给 span 附加键值信息，semantic attributes 用来在系统之间统一这些字段命名。
8. event 可以看作 span 生命周期里的“原始日志”；status 用来标记 span 是否错误结束；记录异常时，最好和错误状态一起使用。
9. OpenTelemetry Python 默认传播格式是 `W3C Trace Context` 和 `W3C Baggage`。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://opentelemetry.io/docs/concepts/context-propagation/
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://opentelemetry.io/docs/languages/python/instrumentation/
