# title

2026-06-13 为什么现在该补 MCP Tools Contract 到 Python Agent 接入

## original source

- 标题：Tools - Model Context Protocol
- 链接：https://modelcontextprotocol.io/specification/2025-06-18/server/tools
- 来源类型：MCP 官方规范
- 访问时间：2026-06-13

- 标题：Model context protocol (MCP) - OpenAI Agents SDK
- 链接：https://openai.github.io/openai-agents-python/mcp/
- 来源类型：OpenAI Agents SDK 官方文档
- 访问时间：2026-06-13

## why read today

你最近几天连续补了 `trace context`、`manual instrumentation`、`baggage`。这一串内容解决的是“Agent 跑起来以后，怎么观察它”。但对你现在这个转型阶段来说，还差另一半能力：**怎么把外部系统以稳定、可治理、可观测的方式接进 Agent。**

MCP 的 `tools` 正好是这个入口。它不是“再学一个新框架 API”，而是在回答更底层的问题：

- 一个工具暴露给模型时，最小合同到底是什么
- 参数、结果、错误、审批、命名冲突分别落在哪一层解决
- 从协议层走到 Python SDK 时，哪些事由框架帮你做，哪些事必须你自己定边界

这对你这种做过 10 年 Java / Spring / IoT 集成、长期面对 ToB 交付和跨系统联调的人非常关键。因为你真正擅长的从来不是“调通一个 demo”，而是“把外部能力接成可维护的生产合同”。今天这篇就是把这个优势迁到 Agent engineering。

## original-text translation

MCP 的 `tools` 规范把“工具”定义成可被语言模型发现并调用的能力。服务端需要暴露工具列表，每个工具至少要说清楚名字、描述和输入参数的 JSON Schema；如果结果本身有稳定结构，还可以继续声明 `outputSchema`。这意味着工具不是一段随意返回文本的黑盒，而是一个带明确输入输出合同的远程能力。

规范还把调用过程分成几层：先 `tools/list` 发现能力，再 `tools/call` 执行能力；参数错误、未知工具这类问题走 JSON-RPC 协议错误，业务执行失败则通过工具结果里的 `isError` 表达。也就是说，**协议层错误**和**业务层错误**是被故意分开的，这和你以前做系统集成时区分“接口没对上”和“业务执行失败”是同一个思路。

OpenAI Agents SDK 的 MCP 文档则把这套协议进一步落到 Python 工程里。它强调接 MCP 前先选传输方式：托管 MCP、Streamable HTTP、SSE、stdio；然后再决定 schema 要不要转严格模式、工具失败是抛异常还是暴露给模型、多个服务器重名工具怎么处理、是否要加人工审批、是否要通过 `_meta` 传每次调用的额外上下文。也就是说，SDK 并不是替你“隐藏 MCP 复杂度”，而是把真正该做的工程决策暴露给你。

把两篇放在一起看，最关键的一点是：**协议定义“工具合同”，SDK定义“工具接入策略”**。前者决定你和外部系统怎样说清楚，后者决定你在 Python Agent 里怎样把这份合同接得安全、可调试、可演进。

## Chinese deep summary

如果把你未来的 Agent 系统想成一个“会推理的集成平台”，那 MCP `tools` 就是它的接口层合同，而不是简单的插件清单。

第一层要吃透的是：`tool` 的核心不是“让模型能调函数”，而是“让模型拿到一个可约束、可验证、可替换的能力描述”。MCP 规范里最重要的不是 `tools/call` 这个动作本身，而是工具定义里那几个字段背后的架构含义：

- `name` 决定模型看到的调用入口
- `description` 决定模型在什么语义场景下会选它
- `inputSchema` 决定参数边界能不能被约束
- `outputSchema` 决定结果能不能被稳定消费

对正在补 Python 的你来说，这里最值得建立的意识是：**工具接入首先是 schema 设计问题，其次才是代码调用问题。** 你以前在 Spring 里做接口联调，最怕的是“字段名像对了，但语义没对齐”。MCP 里也是一样。一个没有清晰 schema 的 tool，看起来能跑，实际上很难评估、很难回放、很难切换实现、很难稳定交付。

第二层是错误分层。MCP 规范把错误拆成两类，非常适合你用集成工程视角去理解：

1. 协议错误：例如工具名不存在、参数不合法、服务端本身异常。
2. 工具执行错误：例如调用第三方 API 失败、命中业务规则、权限不够。

这件事看似普通，实际上对 Agent 系统极其重要。因为模型面对错误时，恢复策略完全不同：

- 如果是参数 schema 不匹配，重点应该是重试前先修正调用格式。
- 如果是业务失败，重点可能是换工具、降级、请求用户确认，或者给出解释。

如果你把两种错误都粗暴地塞成一段字符串，模型就只能“靠猜”做下一步。对 ToB 交付来说，这会直接伤害稳定性和可解释性。

第三层是 `outputSchema` 的价值。很多人做工具时只关心“模型看得懂文本就行”，但规范明确给了结构化输出的路径。它的意义不是形式主义，而是：

- 客户端可以校验返回值
- 上层代码可以稳定消费
- 评测可以按字段对比，而不是按自然语言猜
- 工具可以从“给模型看的助手文本”升级成“可组合的系统能力”

这和你做企业集成时强调 DTO、契约测试、接口文档一致。对于 Agent engineering，`outputSchema` 其实是在把“工具调用”从 prompt 技巧拉回软件工程。

第四层是 SDK 接入时真正要做的几个工程决策。OpenAI Agents SDK 文档里那几个配置项很值得你重视，因为它们几乎正对应生产接入里最常见的坑：

1. `convert_schemas_to_strict`
这不是小优化，而是在尽量把“宽松 schema”收紧成更利于模型和程序共同消费的合同。对你来说，它像把一份模糊接口文档，尽可能压成更明确的参数定义。

2. `failure_error_function`
这个点非常关键。它决定 MCP 调用失败时，是把错误文本继续暴露给模型，还是直接在程序层抛异常。前者偏 Agent 自恢复，后者偏工程强约束。你以后做客户交付时，不同工具的策略可能要不同。

3. `include_server_in_tool_names`
这是典型的多系统集成问题。多个 MCP server 都可能暴露 `search`、`read_file`、`query` 这种名字。如果不做命名治理，模型层就会出现能力碰撞。这个配置其实就是在工具名层面做命名空间隔离。

4. `require_approval`
这和传统系统里的高风险操作审批是同一个思路。删除、写入、外发、支付之类能力，不应该只因为“模型会调”就默认放开。MCP 接入不是功能问题，首先是权限问题。

5. `tool_meta_resolver`
这个配置非常贴近你最近补的 trace / baggage / tenant 上下文。它让你在每次工具调用前，把租户、来源、运行上下文之类信息注入 `_meta`。这意味着你不仅能“调工具”，还能让工具知道“这是谁、从哪来、属于哪次运行”。对后续排障、审计、路由都很有价值。

第五层是今天最重要的迁移认知：**Python Agent 工程并不要求你放弃原有的集成设计思维，反而要求你把它做得更细。**

你过去在 Java / Spring / IoT 集成里熟悉的是这些事：

- 接口合同清不清楚
- 参数是否可校验
- 错误是否分层
- 高风险调用是否要审批
- 上下文是否能跟着请求走
- 多个外部系统接进来后是否会命名冲突

今天 MCP + Agents SDK 只是换了语境，没有换问题本质。你不是从零学一个“AI 新玩法”，而是在把老本行升级成适合 Agent runtime 的版本。

## 3 key takeaways

1. MCP `tools` 的本质是远程能力合同，不是“模型能调的函数列表”；真正决定质量的是 `inputSchema`、`outputSchema`、错误分层和调用边界。
2. OpenAI Agents SDK 没有抹平这些工程决策，而是把它们显式暴露成接入选项：传输方式、严格 schema、失败处理、审批策略、命名空间和 `_meta` 注入都需要你主动设计。
3. 你过去做 Java / Spring / IoT 集成时那套“合同优先、权限优先、可观测优先”的思维，在 Agent engineering 里依然成立，只是接口从 REST / MQ 扩展成了 MCP。

## relation to Agent engineering

这篇内容和 Agent engineering 的关系非常直接，因为它刚好站在“模型推理”和“外部系统能力”之间。

第一，它会影响你今后如何设计自建工具和 MCP server。你会更清楚：什么时候应该把能力做成纯函数工具，什么时候应该返回结构化结果，什么时候应该要求人工审批，什么时候应该把运行上下文通过 `_meta` 带过去。

第二，它会影响你如何做 eval。很多 Agent 评测失败，不是模型不会思考，而是工具合同太松：参数约束不清、错误格式混乱、结果不可稳定解析。你把工具合同收紧以后，评测才会从“感觉这次答得不错”变成“这次 tool 调用哪里出错、为什么出错、能否自动恢复”。

第三，它会影响你怎样搭 Java 和 Python 的桥。未来很可能不是“全部重写成 Python”，而是 Python Agent 调用 MCP server，再去接你熟悉的 Java 服务、老系统 API、企业内网能力。理解工具合同和接入策略之后，你会更容易把旧能力包装成新 Agent 可用的接口层，而不是推倒重来。

## a small action for tonight

今晚做一个 40 分钟以内的小练习，不求接真实平台，只求把“合同”和“策略”分清：

1. 任选一个你熟悉的企业能力，比如“查询设备状态”或“按租户搜索工单”，先只写工具定义，不写实现。
2. 给它补齐 `name`、`description`、`inputSchema`、`outputSchema`，并明确一类协议错误和两类业务错误。
3. 再假设它接到 OpenAI Agents SDK 里，补一页简单设计说明：
是否开启 `convert_schemas_to_strict`
失败时是抛异常还是返回模型可见错误
是否需要 `require_approval`
是否要通过 `_meta` 传 `tenant_id`、`conversation_id`、`trace_id`
4. 最后问自己一个问题：如果半年后这个 tool 要从 Python 实现换成 Java 服务实现，当前这份合同是否还能不改上层 Agent 提示词和评测用例？

## 原文关键段落翻译（人工翻译，放在文末）

1. MCP 允许服务器暴露可被语言模型调用的工具；工具用来和外部系统交互，比如查数据库、调用 API 或执行计算，每个工具都由唯一名称和描述其 schema 的元数据标识。
2. 发现工具时，客户端通过 `tools/list` 获取可用能力；真正执行时，再通过 `tools/call` 按工具名和参数发起调用。
3. 工具定义里除了名称、描述和输入 schema，还可以带 `outputSchema` 与行为注解；如果带了 `outputSchema`，服务端必须返回符合该结构的结果，客户端则应该校验。
4. 工具错误分两层：未知工具、参数非法这类问题属于协议错误；API 失败、业务规则失败这类问题则通过工具结果中的 `isError` 表达。
5. OpenAI Agents SDK 在接 MCP 时，不只是支持多种 transport，还要求你决定工具 schema 是否转严格模式、失败如何暴露、多个服务器同名工具如何处理。
6. 对本地 MCP server，SDK 还把审批策略、按次调用的 `_meta` 注入、工具过滤和 tracing 这些现实工程问题都放到了显式配置里。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://modelcontextprotocol.io/specification/2025-06-18/server/tools
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.github.io/openai-agents-python/mcp/
