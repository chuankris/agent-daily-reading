# title

2026-05-05 为什么做 Agent 不能只懂 MCP tools，还要补上 resources 和 prompts

## original source

- 标题：Resources | Model Context Protocol Specification 2025-06-18
- 链接：https://modelcontextprotocol.io/specification/2025-06-18/server/resources
- 来源类型：官方规范
- 规范版本：2025-06-18
- 访问时间：2026-05-05

- 标题：Prompts | Model Context Protocol Specification 2025-06-18
- 链接：https://modelcontextprotocol.io/specification/2025-06-18/server/prompts
- 来源类型：官方规范
- 规范版本：2025-06-18
- 访问时间：2026-05-05

## why read today

你昨天刚补完 `tool calling + MCP tools contract`，如果今天继续只盯着“工具怎么调”，会很容易把 MCP 理解成“给模型挂一堆函数”。这对做 Demo 够了，但对你这种做了 10 年 Java / Spring / IoT 集成 / ToB 交付的人来说还不够，因为真正的工程问题通常不是“能不能调”，而是：

1. 哪些上下文应该由系统稳定提供，而不是每轮 prompt 手工拼接；
2. 哪些能力应该让用户显式触发，而不是让模型偷偷决定；
3. 工具、资源、提示模板这三类东西，到底怎么分工，才不会把 Agent 接口面做成一锅粥。

MCP 里的 `resources` 和 `prompts`，就是补这两个空白的。前者解决“可被引用的上下文资产如何标准暴露”，后者解决“可复用的人机交互模板如何标准暴露”。把它们和 `tools` 放在一起看，你才会真正理解 MCP 不只是函数调用协议，而是一整套 Agent 能力面协议。

## original-text translation

MCP `resources` 规范先把定位说得很清楚：服务器可以向客户端暴露资源，这些资源为语言模型提供上下文，例如文件、数据库 schema 或应用专属信息；每个资源都由一个 URI 唯一标识。规范特别强调，resources 的使用模型是 application-driven，也就是由宿主应用决定如何把这些资源纳入上下文。协议举的例子包括：在 UI 里让用户显式选择资源、提供搜索和过滤、或者由应用按启发式规则自动纳入上下文，但协议本身不强制具体交互形式。

在能力与消息层面，支持 resources 的 server 必须声明 `resources` capability，还可以选择是否支持 `subscribe` 和 `listChanged`。客户端用 `resources/list` 发现资源，用 `resources/read` 读取内容；如果 server 提供参数化资源，还可以通过 `resources/templates/list` 暴露 URI 模板。资源内容既可以是文本，也可以是二进制 blob；资源、模板和内容块还可以带注解，例如 `audience`、`priority`、`lastModified`，帮助客户端决定哪些内容更适合展示给用户，哪些更适合优先放进上下文。

`prompts` 规范则定义了另一类能力：server 可以暴露 prompt templates，让客户端发现、获取并按参数定制这些模板。这里的交互模型和 resources 不一样，规范明确说 prompts 是 user-controlled，设计意图是让用户可以显式选择使用，典型形式是 UI 里的 slash command。支持 prompts 的 server 必须声明 `prompts` capability；客户端用 `prompts/list` 查看可用模板，用 `prompts/get` 取回某个模板展开后的消息内容。

规范里的 prompt 结果不是单纯一段字符串，而是一组 message。每条 message 带 `role` 和 `content`，内容可以是 text、image、audio，甚至直接嵌入 resource。这意味着 prompt 在 MCP 里不是“给模型一段固定文案”，而是“给客户端一组结构化消息模板”，它既可以复用参数，也可以把服务器管理的文档、代码样例或其他参考资料直接嵌进交互流里。规范同时要求做参数校验、能力协商，并提醒要防 prompt injection 和未授权资源访问。

## Chinese deep summary

如果把今天这两份规范压成一句工程话，那就是：`tools` 负责“做事”，`resources` 负责“供上下文”，`prompts` 负责“组织交互入口”。

很多人第一次接触 MCP，会天然把注意力全放在 tools 上，因为 tools 最像“模型会调用的函数”。但从系统设计角度看，只靠 tools 会带来一个很常见的问题：你会把本该稳定暴露的上下文、以及本该由用户选择的交互入口，统统硬塞进工具描述或系统 prompt 里。久了以后，整个 Agent 能力面会变得模糊、难治理、难观测。

这正是今天补 `resources` 和 `prompts` 的意义。

先说 `resources`。
它解决的不是“让模型多一个动作”，而是“让系统多一层可被标准引用的上下文资产”。这对你这种做过大量企业集成的人很重要，因为你以前就很熟悉一个事实：很多系统集成并不是缺动作 API，而是缺稳定、可枚举、可筛选、可缓存的上下文输入面。

举个贴近你背景的类比：

1. `tools` 像“调用工单系统创建工单”；
2. `resources` 像“当前客户站点清单、设备台账、部署拓扑、数据库 schema、配置模板”；
3. `prompts` 像“为某类工作准备好的标准操作入口”，例如“巡检总结”“代码评审”“事故复盘提纲”。

如果没有 `resources`，很多团队会怎么做？最常见的做法是把这些上下文临时塞进 prompt，或者让工具顺手返回一大段说明文本。短期看很省事，长期看问题很多：

1. 上下文资产没有统一标识，无法稳定引用；
2. 用户和应用都很难知道“到底有哪些可用上下文”；
3. 资源更新后没有一致的刷新机制；
4. 相同内容会在多个 prompt、多个工具描述、多个系统消息里反复复制。

MCP 的 `resources/list`、`resources/read`、`resources/templates/list`、`subscribe`、`listChanged`，其实就是在把这件事协议化。它告诉你：上下文资产不是随手拼出来的一段字符串，而是可以被发现、读取、订阅、增量感知的对象。

这一点和你做 IoT / ToB 交付时的思路非常一致。你以前不会把“设备清单”“点位定义”“站点配置”埋在代码注释里，而会把它们做成可以查、可以版本化、可以对接的资产。到了 Agent 工程里，`resources` 本质上也是这件事，只是面向的是模型上下文层。

再说 `prompts`。
很多初学者会把 prompt 理解成“开发者自己写的一段提示词”。但 MCP 里的 `prompts` 更接近“服务器提供的、可参数化的交互模板”。它的重点不是“把一段话发给模型”，而是把某种交互模式做成可发现、可调用、可填参、可治理的标准入口。

为什么这对你现在特别重要？因为你正在从“后端系统集成”转向“Agent 工程”，而这类转型最容易忽略的一件事，就是把用户操作入口和模型执行能力混在一起。比如：

1. “帮我做代码评审”是一个交互入口，不是一个工具；
2. “读取某个仓库文件”是一个资源读取或工具动作；
3. “生成一份巡检日报”可以是一个 prompt 模板，内部再引用资源、触发工具。

如果不分层，最后常见的结果是：什么都做成工具，或者什么都塞进一个大系统 prompt。前者让用户看不到清晰入口，后者让运行时难以复用和审计。

`prompts` 规范里最值得你记住的，是它明确把这类能力设计成 `user-controlled`。这句话很有工程味。它意味着某些模板不是给模型自己偷偷挑的，而是给用户显式触发的。对企业场景尤其重要，因为用户经常需要知道“我现在执行的是哪一类工作流入口”，而不是只看到一个模糊的聊天框。

还有一个容易被忽略但很关键的点：`prompts/get` 返回的是消息数组，而不是一整块纯文本。这件事非常重要，因为它让 prompt 模板可以天然承载更复杂的交互结构：

1. 可以有不同 `role` 的消息；
2. 可以带文本、图片、音频；
3. 可以直接嵌入 resource；
4. 可以在模板展开阶段完成参数填充。

这说明 MCP `prompts` 不是“prompt 仓库”，而是“结构化消息模板层”。对后面做 Agent UI、Slash Commands、面向用户的任务入口，都很有价值。

把三者放在一起看，MCP 的工程分层就清楚了：

1. `resources` 回答“上下文资产是什么，怎么发现，怎么读取，怎么跟踪变化”；
2. `prompts` 回答“用户可触发的交互入口是什么，模板如何展开成结构化消息”；
3. `tools` 回答“系统动作如何被发现、调用、返回结果和报错”。

这套分层为什么比“全塞进 prompt”更高级？因为它让每一类东西都回到更合适的生命周期里：

1. 资源有自己的 URI、元数据、更新通知；
2. Prompt 有自己的参数、展示名、可选说明、消息结构；
3. 工具有自己的 input schema、output schema、执行错误语义和安全边界。

对于你的学习阶段，这个理解比继续堆更多框架名更值钱。因为你后面无论用 Python、FastAPI、Spring AI 还是 MCP SDK，本质都绕不过这个能力分层。谁负责给上下文，谁负责给入口，谁负责做动作，这三个问题一旦分清，系统就容易稳；分不清，后面 Eval、Trace、权限、成本都会跟着混乱。

## 3 key takeaways

1. `resources` 不是工具的附属品，而是 MCP 中独立的上下文资产层，适合承载文件、schema、配置、知识片段这类可被稳定引用的内容。
2. `prompts` 不是普通提示词文本，而是 user-controlled 的结构化交互模板；它更像面向用户暴露的任务入口，而不是模型私下决定的动作。
3. 把 `resources / prompts / tools` 分层看，才能把 Agent 从“聊天框 + 一堆函数”推进到“有上下文面、有入口面、有执行面”的可治理系统。

## relation to Agent engineering

这篇内容和你接下来补 Python / Agent / MCP 的方式直接相关。

第一层是接口设计习惯。你以后如果做企业内 Agent，不要一上来就把所有能力都注册成工具。先判断这到底是“上下文资产”“用户入口”还是“执行动作”。这个分类做对了，系统复杂度会低很多。

第二层是宿主应用职责。`resources` 是 application-driven，`prompts` 是 user-controlled，这两个词背后其实都在强调：宿主应用不能退位。应用要决定资源怎么选、怎么订阅、什么时候自动带入；也要决定哪些 prompt 以 slash command、按钮或菜单暴露给用户。

第三层是后续 Eval 与 Trace。以后你遇到 Agent 表现不稳时，要先分辨问题是出在上下文供给、交互入口还是动作执行。比如“回答偏题”可能是资源选择不对；“用户体验差”可能是入口设计不清；“执行失败”才可能是 tool 调用本身的问题。

第四层是和 Java/Spring 经验对接。你可以把 `resources` 类比成可管理的配置与文档资产，把 `prompts` 类比成业务操作模板或向导入口，把 `tools` 类比成后端动作接口。这个类比一旦建立，你学 MCP 就不会只停在 Python Demo 视角。

## a small action for tonight

今晚花 30 分钟，做一个“企业 Agent 能力分层草图”：

1. 选一个你熟悉的场景，例如“客户站点巡检助手”或“交付问题排查助手”。
2. 先列 5 个应该做成 `resources` 的对象，例如站点清单、设备台账、部署图、告警规则、数据库 schema。
3. 再列 3 个应该做成 `prompts` 的入口，例如“生成巡检总结”“解释某条告警”“整理客户沟通纪要”。
4. 最后列 3 个真正应该做成 `tools` 的动作，例如创建工单、查询实时状态、触发诊断脚本。

只要今晚把这三列列清楚，你对 MCP 的理解就会比“会不会写一个 tool demo”更接近工程实战。

## 原文关键段落翻译（人工翻译，放在文末）

1. `resources` 规范把资源定义为服务器暴露给客户端、供语言模型使用的上下文数据，例如文件、数据库 schema 或应用专属信息，并要求每个资源都有唯一 URI。
2. `resources` 的交互模型是 application-driven，意思是宿主应用决定如何把资源纳入上下文，协议不强制具体 UI 或选择方式。
3. 支持 resources 的服务器必须声明 `resources` capability；客户端通过 `resources/list` 发现资源，通过 `resources/read` 读取资源，还可以通过模板列表和订阅机制处理参数化资源与资源更新。
4. `prompts` 规范把 prompt templates 定义为服务器暴露给客户端的结构化消息模板，客户端可以发现、获取，并通过参数进行定制。
5. `prompts` 被明确设计成 user-controlled，通常由用户在界面中显式触发，例如 slash command，而不是默认交给模型自行决定何时使用。
6. `prompts/get` 返回的是消息数组，每条消息包含角色与内容；内容不仅可以是文本，也可以是图片、音频或嵌入资源，因此 prompt 在 MCP 中是结构化交互层，而不是单段文本。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://modelcontextprotocol.io/specification/2025-06-18/server/resources
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://modelcontextprotocol.io/specification/2025-06-18/server/prompts
