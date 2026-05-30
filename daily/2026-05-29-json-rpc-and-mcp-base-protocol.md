# title

2026-05-29 为什么先补 `JSON-RPC 2.0`，再看 MCP，能更快把 Agent 工程做扎实

## original source

- 标题：`JSON-RPC 2.0 Specification`
- 链接：https://www.jsonrpc.org/specification
- 来源类型：JSON-RPC Working Group 官方规范
- 访问时间：2026-05-29

- 标题：`Overview - Model Context Protocol`
- 链接：https://modelcontextprotocol.io/specification/2025-11-25/basic/index
- 来源类型：MCP 官方规范
- 访问时间：2026-05-29

## why read today

你这轮补课已经把 MCP 的生命周期、resources、prompts、roots、sampling、authorization 这些上层概念补过了，但如果底层消息模型没有彻底吃透，后面一旦进入真实工程，你会很容易在三个地方吃亏：

1. 调试时分不清“这是业务错误、协议错误，还是 transport 错误”。
2. 自己封装 tool / MCP client / MCP server 时，消息边界写得像 REST，却不是协议实现。
3. 看到 SDK 能跑，就误以为自己理解了 MCP，结果一换语言、一换 transport、一做网关就开始断。

今天这篇的价值就在这里：它不追新功能，而是回到底层公共契约。

你过去做过 10 年 Java / IoT 集成和 ToB 交付，应该很熟悉一件事：真正决定系统是否能长期维护的，往往不是“业务字段有多少”，而是“协议层有没有最小且稳定的共同约束”。MCP 对 Agent 世界的意义，很像你以前做设备接入、平台集成时看到的那类统一协议层；而 `JSON-RPC 2.0`，就是这层协议下面最值得先补牢的那块地基。

如果你把这层看透，后面再看 Python SDK、Java SDK、MCP gateway、agent host、tool executor，就不会只看到“调用工具”，而会开始主动检查：

1. 这是不是 request、response、notification 其中一种？
2. `id` 的语义有没有被正确维护？
3. 出错时是 JSON-RPC error，还是应用自己的业务错误塞进 `result` 里了？
4. 批量消息、单向通知、会话内唯一 ID 这些约束有没有被破坏？

这组判断能力，比继续追一个新框架名字更值。

## original-text translation

`JSON-RPC 2.0` 规范先把它自己定义得很克制：它是一个无状态、轻量级、与具体传输无关的远程过程调用协议。规范核心不是业务语义，而是几种基础数据结构及其处理规则。它强调自己可以跑在同进程、socket、HTTP 或各种消息环境之上，数据格式使用 JSON。

规范把请求对象说得很清楚：一次 RPC 调用，就是向服务端发送一个 request object。这个对象必须带上 `jsonrpc: "2.0"`，必须有 `method`，可以有 `params`，如果带了 `id`，服务端回复时就必须带回同一个 `id`。这里的 `id` 不是普通字段，而是客户端建立的关联标识，用来把请求和响应对上。

规范还专门区分了 notification：如果一个请求没有 `id`，它就不再是“等回复”的调用，而是通知。通知的语义是发送方不关心对应响应，因此服务端不得回复。换句话说，`有没有 id` 不只是序列化细节，而是消息语义本身的一部分。

参数结构也被压得很简单：`params` 如果存在，必须是结构化值，要么是按位置的数组，要么是按名字的对象。这意味着协议层只约束“结构”，不替应用决定具体业务模型。

在响应对象上，规范要求同样严格：成功时必须有 `result`，失败时必须有 `error`，二者不能同时出现；响应里的 `id` 必须与请求一致。如果请求本身坏到连 `id` 都识别不出来，例如 JSON 解析错误或非法 request，对应错误响应里的 `id` 才允许是 `null`。

`error` 对象也有最小公共结构：必须有整数类型的 `code` 和简短的 `message`，可选附加 `data`。规范预留了一段标准错误码区间，例如解析错误、非法请求、方法不存在、参数非法、内部错误。这意味着跨语言实现时，最底层错误至少有一组共同语言。

批量请求则进一步说明了这个协议的调度哲学：客户端可以把多个 request 放进一个数组一起发，服务端可以并发处理，返回顺序也不要求和请求顺序一致，客户端应靠 `id` 去做配对。这里很关键的一点是，notification 即使出现在 batch 里，也不应该收到响应。

MCP 官方规范在这个基础上又加了一层明确约束：所有 MCP client 和 server 之间的消息，都必须遵循 `JSON-RPC 2.0`。它把消息分成三种：request、response、notification，并继续细化成 MCP 自己的要求。

例如在最新 `2025-11-25` 版本里，MCP 明确要求 request 的 `id` 必须是字符串或整数，不能是 `null`，而且在同一个 session 里不能重用。response 必须带和请求相同的 `id`；成功必须放 `result`，失败必须放 `error`。notification 依然是单向消息，因此不得包含 `id`。

MCP 还强调它自己的协议分层：基础协议只解决核心 JSON-RPC 消息类型；其上再叠加生命周期管理、HTTP transport 上的 authorization、server features、client features 和 utilities。换句话说，tools、resources、sampling 这些你现在更常看到的能力，都不是“裸奔”的业务特性，而是立在一套更底层的消息契约之上。

## Chinese deep summary

如果把今天这两份规范压成一句话，那就是：

**MCP 不是“某种智能工具接口魔法”，它首先是一套建立在 `JSON-RPC 2.0` 之上的严格消息协议。**

第一层，你要先把 MCP 从“LLM 工具调用”降维回“协议工程”。

很多人第一次接触 MCP，会直接盯着 `tools/list`、`tools/call`、`resources/read` 这些能力看，于是脑子里默认它像一个给模型用的 REST API。这个理解很危险。因为一旦你把 MCP 当 REST，你就会自然犯几类错误：

1. 用 URL 路由思维替代方法名思维。
2. 用 HTTP 状态码语义替代协议错误对象语义。
3. 把一次消息当成“接口请求”，而不是“会话中的协议帧”。
4. 忽视 notification 这种单向消息，结果把本该无响应的事件流做错。

对你这种做过系统集成的人，这个区分尤其重要。你过去应该见过很多“看起来像 HTTP，实际上有自己消息语义”的系统。MCP 也是同类问题。HTTP、stdio、SSE、streamable HTTP 只是承载层；真正决定互操作性的，是上面那层 JSON-RPC 消息模型。

第二层，`id` 是 MCP/JSON-RPC 世界里的因果锚点，不是一个可有可无的流水号。

这点很适合用你熟悉的 Java 集成经验来理解。以前你在做异步回调、设备上报、MQ 请求应答时，最怕的不是“没字段”，而是“关联字段失真”。JSON-RPC 的 `id` 就是这个关联锚点。

在基础 JSON-RPC 里，`id` 决定请求和响应如何配对；在 MCP 里，这个约束更严：同一 session 内不能重用已经发过的 request ID。这个要求背后不是教条，而是为了让实现可以在一个连接里安全地并发跑多个中的操作。

对 Agent 工程来说，这个点非常现实。因为 agent host 不会只发一个请求，它会同时处理：

1. 初始化协商
2. 工具枚举
3. 资源读取
4. 进度通知
5. 取消请求
6. 可能还有并发的多工具调用

如果你对 `id` 的理解只是“随便生成个数字”，那一旦切到多线程、异步、重试、代理转发，问题就会立刻冒出来。

第三层，notification 是很多实现最容易写错、但最能暴露协议素养的地方。

为什么？因为业务开发者天然习惯“请求必有响应”。但 JSON-RPC 明确说了：没有 `id` 的就是 notification，而 notification 不应该收到响应。MCP 继承了这件事。

这对 Agent 工程尤其关键，因为很多运行时事件本来就是单向的，比如：

1. 日志/进度上报
2. 某些状态变更通知
3. 订阅式更新
4. 不要求即时确认的事件消息

如果你把所有东西都做成 request/response，短期看似统一，长期会把协议层搞得又重又乱。你会平白制造无意义的等待、重试和错误处理逻辑。

第四层，JSON-RPC 的 `error` 语义和业务失败语义，必须在脑子里分层。

这是最容易在工程里混掉的一点。协议错误表示“你这条消息本身有问题”或者“协议层无法完成这次调用”，例如：

1. JSON 解析失败
2. request object 非法
3. 方法不存在
4. 参数结构不合法
5. 协议处理内部错误

但业务失败未必是协议失败。比如某个 MCP tool 正常被调用了，只是查询结果为空、外部服务超时、目标对象不存在。这些应该如何表达，要看上层 contract，而不是一股脑都映射成 JSON-RPC 标准错误码。

这个分层观念对你很关键。因为你过去做企业集成时，应该已经吃过类似亏：传输层、协议层、业务层、领域层的错误一旦揉在一起，后面排障就会变成灾难。Agent 工程也完全一样。真正专业的实现，不是“能把异常吐出来”，而是“错误边界划得清楚”。

第五层，MCP 在 JSON-RPC 之上做的事，不是推翻，而是收紧和模块化。

MCP 没有重新发明 request/response/notification，而是在已有消息语义上增加适合 Agent 生态的工程约束。最典型的几个是：

1. 同 session 内 request ID 不可重用。
2. 基础协议和生命周期管理是所有实现都必须支持的最小集合。
3. tools、resources、sampling、authorization 都是建立在基础协议之上的模块能力。
4. JSON Schema 成为很多参数和结构验证的共同底座。

这背后是一种很成熟的协议设计思路：先收紧最小公共层，再往上扩展能力，而不是一开始就把全部业务特性揉进通信层。对于你这种从 Java/Spring 后端转来的人，这其实是熟悉的味道，和很多企业集成协议、网关规范、中间件契约的设计哲学是相通的。

第六层，今天这篇材料最适合你当前阶段的原因，不是它“新”，而是它能补掉一个会反复绊脚的结构性短板。

你现在最容易出现的学习错位，不是不会写 Python，而是：

1. 会跟着 SDK 跑样例
2. 但看不清底层协议约束
3. 一旦换语言、换 transport、换运行时边界，就不知道哪里该自己兜底

把 JSON-RPC 和 MCP 基础协议读透后，你后面看任何 SDK，都会多一层“拆框架”的能力。你会开始主动问：

1. SDK 帮我维护了哪些 `id`、session、error 规则？
2. 哪些是协议强制要求，哪些只是这个 SDK 的实现选择？
3. 如果我自己写 gateway / adapter / bridge，最小正确行为应该是什么？

这正是从“会用框架”走向“能写可靠 Agent 基础设施”的分水岭。

## 3 key takeaways

1. `JSON-RPC 2.0` 定义的是跨语言、跨传输的最小消息契约：request、response、notification、`id` 关联、`result/error` 互斥、标准错误对象和 batch 行为。
2. MCP 不是替代 `JSON-RPC`，而是在其上增加更严格的会话约束和模块分层；因此理解 MCP，先理解底层消息模型，比先背 API 名字更重要。
3. 对 Agent 工程来说，很多线上问题本质不是“模型不会调工具”，而是协议边界、错误分层、并发关联和单向通知语义没有实现正确。

## relation to Agent engineering

这篇内容和 Agent engineering 的关系很直接，而且正好对准你当前的 Python / MCP / Eval 缺口。

第一，它会直接提升你读 SDK 源码和排查 wire message 的能力。以后你再看 Python MCP SDK、Java SDK 或 host runtime 的日志，不会只盯着业务参数，而会先检查消息类型、`id`、`result/error`、notification 语义是不是对的。

第二，它能帮你更稳地设计 tool gateway、MCP adapter 和系统集成桥。你过去做过很多 ToB 交付，这类桥接层往往才是最容易积累技术债的地方。协议层一旦想清楚，后面的 Python 实现反而只是细节。

第三，它会让你更容易建立“协议错误”和“评估失败”之间的边界。不是所有失败样本都该进模型评估，有些失败其实是 request 结构错了、session 处理错了、或者 notification/response 实现错了。把这层分开，eval 才不会被脏数据污染。

第四，它能把你原本的 Java / Spring / IoT 集成经验重新利用起来。你不是从零学“智能应用”，而是在把熟悉的协议设计、消息关联、错误分层、可观测性思维，迁移到 Agent runtime 和 MCP 生态里。

## a small action for tonight

今晚做一个 40 分钟的小实验，不追功能多，只追协议感觉变清楚：

1. 用 Python 写一个最小 JSON-RPC server，只支持一个 `echo` 或 `sum` 方法。
2. 手写三种消息发过去：正常 request、notification、非法 request。
3. 观察服务端分别应该返回什么，特别是 notification 不应回包、非法 request 应返回标准 error。
4. 再把这个最小 server 改造成“像 MCP 一样”的 method 命名，例如 `tools/list`、`tools/call`，先不接真实模型，只保留协议壳子。
5. 最后打印一份你自己的检查清单：`id` 是否唯一、`result/error` 是否互斥、业务失败是否错误地塞成协议错误、notification 是否被误回复。

你做完这一轮，再回头看 MCP SDK，就不再只是“会调用”，而是已经能看出它在替你维护哪些协议不变量。

## 原文关键段落翻译（人工翻译，放在文末）

1. `JSON-RPC` 是一种无状态、轻量级、与传输无关的远程过程调用协议；规范核心定义的是数据结构以及处理这些结构的规则。
2. 一个 request object 必须带 `jsonrpc: "2.0"` 和 `method`；如果包含 `id`，服务端回复时必须带回同一个 `id`，用来建立请求与响应之间的关联。
3. 没有 `id` 的 request 就是 notification；notification 表示客户端不关心对应响应，因此服务端不得返回响应。
4. 成功响应必须包含 `result`，失败响应必须包含 `error`；这两个字段不能同时出现。
5. 标准错误对象至少包含整数类型的 `code` 和简短的 `message`，并预留了一组通用错误码用于解析错误、非法请求、方法不存在、参数错误和内部错误。
6. 批量请求可以并发处理，返回顺序不要求和请求顺序一致，客户端应通过 `id` 而不是数组位置去匹配响应。
7. MCP 官方规范要求所有 client 与 server 之间的消息都遵循 `JSON-RPC 2.0`，并把消息分成 request、response 和 notification 三类。
8. 在 MCP `2025-11-25` 规范里，请求 `id` 必须是字符串或整数，不能是 `null`，并且在同一个 session 内不得重用。
9. MCP 把基础协议和生命周期管理定义为所有实现都必须支持的最小集合，其上的 tools、resources、authorization 和 utilities 都建立在这层公共契约之上。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://www.jsonrpc.org/specification
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://modelcontextprotocol.io/specification/2025-11-25/basic/index
