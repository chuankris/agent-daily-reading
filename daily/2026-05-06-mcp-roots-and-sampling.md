# title

2026-05-06 为什么做 MCP 集成不能只看 server capabilities，还要补上 `roots` 和 `sampling`

## original source

- 标题：Roots | Model Context Protocol Specification 2025-06-18
- 链接：https://modelcontextprotocol.io/specification/2025-06-18/client/roots
- 来源类型：官方规范
- 规范版本：2025-06-18
- 访问时间：2026-05-06

- 标题：Sampling | Model Context Protocol Specification 2025-06-18
- 链接：https://modelcontextprotocol.io/specification/2025-06-18/client/sampling
- 来源类型：官方规范
- 规范版本：2025-06-18
- 访问时间：2026-05-06

## why read today

你这几天已经把 MCP 的 `tools`、`resources`、`prompts` 补上了，如果现在停住，很容易形成一个偏差：以为 MCP 主要是在定义“server 能给模型什么能力”。这只看到了半边。

对你这种做了 10 年 Java / Spring / IoT 集成 / ToB 交付的人来说，真正更接近工程现实的问题其实是另外两类：

1. 文件、仓库、工作目录这些边界，到底由谁告诉 server，怎么限制范围；
2. 真正发起模型调用的控制权，到底在 server 手里，还是在宿主应用 / 客户端手里。

`roots` 和 `sampling` 恰好回答这两个问题。`roots` 解决“server 在文件系统里能看到哪一圈”，`sampling` 解决“server 想借客户端去调模型时，权限和选择权在哪里”。如果不把这两个 client-side feature 补上，你对 MCP 的理解还是会停在“server 暴露接口”，还没进入“宿主应用怎么掌权”这一层。

## original-text translation

`roots` 规范先把定位说得很直接：MCP 提供一种标准方式，让客户端把文件系统中的“根目录”暴露给服务器。根目录定义了服务器可操作的边界，让服务器知道自己可以访问哪些目录和文件。支持 `roots` 的客户端要在初始化时声明 capability；服务器通过 `roots/list` 请求获取当前 roots 列表，如果客户端支持 `listChanged`，当根目录集合变化时，还要发送 `notifications/roots/list_changed` 通知。

规范还强调了两个很重要的工程含义。第一，`roots` 的交互模型通常通过 workspace 或 project 配置界面暴露给用户，例如让用户选择允许访问的目录；协议本身不强制 UI 形式。第二，安全边界是明确写出来的：客户端必须只暴露具备适当权限的 roots，要校验 root URI，防止路径穿越；服务器也应当尊重 roots 边界，对所有操作路径做校验，并处理 root 变得不可用的情况。

`sampling` 规范则描述了反向的一种能力：不是客户端去调用服务器工具，而是服务器请求客户端代为完成一次 LLM 采样。规范把它定义成“服务器通过客户端向语言模型请求生成”的标准方式，并明确说明这样做可以让客户端保留对模型访问、模型选择和权限的控制，甚至可以做到 server 不持有模型 API key。支持 sampling 的客户端必须声明 `sampling` capability；服务器通过 `sampling/createMessage` 发起请求，参数里可以带 messages、modelPreferences、systemPrompt、maxTokens，以及是否要包含来自 MCP server 的上下文。

规范对 `sampling` 的人机控制要求也写得很清楚。开始采样前，客户端应该让用户检查请求并决定是否批准；返回模型生成结果前，也应该让用户检查响应再决定是否给 server 看。模型选择上，server 只能给出 preference 和 hint，真正选哪个模型由客户端决定；即使 server 想要某个模型名，客户端也可以映射到自己可用的同类模型。规范还要求两边处理敏感数据、做内容校验和限流。

## Chinese deep summary

如果把今天这两份规范压成一句工程话，就是：MCP 不只是 server 把能力“挂出来”，还要求 client 明确声明“我愿意给你哪些文件边界，以及我愿意替你调用哪些模型能力”。

很多人学 MCP 时会天然盯着 server feature，因为 `tools`、`resources`、`prompts` 都比较像“我能提供什么”。但真正做成可交付系统时，宿主应用和客户端不能退位。你以前做企业集成很熟悉一个现实：能不能接入，不只取决于下游系统暴露了什么接口，也取决于网段边界、目录权限、凭证归属、审批流程和操作入口。MCP 的 `roots` 和 `sampling`，本质上就是把这两类“客户端主权”协议化了。

先说 `roots`。

它看起来像一个很小的能力，只是列一下目录，但它背后其实在回答一个特别核心的问题：server 到底在“哪片工作空间里”做事。对你这种长期做交付和集成的人，这个问题一点都不小。因为很多实际事故都不是“功能不会用”，而是边界没收住。

如果没有 `roots` 这层约束，最常见的退化方式有两种：

1. 默认把整台机器或整块工作区都暴露给 server；
2. 让 server 自己猜工作目录，再靠实现细节去兜路径限制。

这两种做法都不稳。前者权限面太大，后者责任边界不清。`roots` 规范的价值就在于，它把“允许访问哪些目录”提升成了显式协商对象，而不是隐藏前提。server 不该假设自己天然能读所有文件；它应该通过 `roots/list` 拿到边界，并在这个边界内工作。

这点和你以前做 IoT / ToB 集成时非常像。你不会默认一个集成服务能访问客户所有库表、所有设备、所有文件共享；你会先明确租户边界、站点边界、网段边界。`roots` 在 MCP 里干的就是同类事情，只不过对象换成了“文件系统上下文边界”。

再往前走一步，`roots/list_changed` 也值得你记。它说明这个边界不是永久静态常量，而是可能变化的运行时信息。比如用户切换了 workspace、关闭了某个仓库、或者重新授权了一个目录。对工程实现来说，这意味着 server 不能只在启动时读一次 roots 然后永远假设不变，而应该把它当成可更新的能力输入。这种思路和你做配置热更新、租户路由变更、设备在线集变化，其实是同一类运行时治理问题。

再说 `sampling`。

如果只用一句话讲它，最关键的不是“server 也能让模型继续想”，而是“真正的模型访问控制权仍在 client / host 侧”。这一点对 Agent 工程特别关键，因为很多人会误把 MCP server 理解成“自己也能随便调模型的迷你 Agent”。规范不是这么设计的。

`sampling` 其实是在建立一种受控反向调用：

1. server 提出“我希望有一次模型生成”；
2. client 决定是否批准；
3. client 决定用哪个模型；
4. client 决定是否允许带上哪些上下文；
5. client 在把生成结果返回给 server 之前，还可以再做人机确认。

这和你以前做企业系统时的“外部系统发起代办请求，但最终权限和资源消耗由平台统一控制”非常像。换成 Agent 语境，就是：MCP server 可以有 agentic behavior，但宿主应用不应该把模型预算、模型权限、敏感提示词和最终输出审查全部外包给 server。

这一点正好补上你前几天读的 `tools / resources / prompts`。前面那几篇更偏“server 提供什么”，今天这篇补的是“client 保留什么”。两边合起来，MCP 的真实权责结构才完整：

1. `tools` 负责 server 暴露动作；
2. `resources` 负责 server 暴露上下文资产；
3. `prompts` 负责 server 暴露用户入口模板；
4. `roots` 负责 client 告诉 server 文件系统边界；
5. `sampling` 负责 client 决定 server 是否能借自己调模型。

这套分层为什么重要？因为它直接影响你以后怎么做企业 Agent 的治理。

如果没有 `roots`，文件权限和工作区边界会很快变成一团隐式假设，后面出了问题你很难追责和排查。

如果没有 `sampling` 的主权意识，你就容易把“server 有推理能力”误解成“server 应该持有模型 key、直接无限制调模型”。这在 Demo 里方便，在正式系统里往往是成本、审计和安全风险。

尤其对你这种 Java / Spring / 交付背景的人，今天最值得建立的不是 API 细节记忆，而是这个判断：MCP 的宿主应用不是一个透明转发层，而是治理层。它既不是被动地把所有 server 能力展示出来，也不是盲目代 server 调模型，而是负责边界、授权、审批、模型选择和上下文汇聚。

再把它翻成更贴近你背景的话：

1. `roots` 很像“客户现场允许接入的目录 / 盘符 / 仓库白名单”；
2. `sampling` 很像“平台统一代调大模型的网关与审批面”；
3. host / client 很像“你以前做集成平台时那个真正负责权限、配额、审计和路由的总控层”。

一旦这样类比，你学 MCP 就不会停在 Python demo 视角，而会更快进入可落地的 Agent 平台视角。

## 3 key takeaways

1. `roots` 不是附属细节，而是 MCP 明确的文件系统边界协商机制；server 应在 roots 范围内工作，而不是默认拥有整个工作区。
2. `sampling` 的核心不是“让 server 多一次生成”，而是“让 client 保留模型访问、模型选择、用户审批和结果放行的控制权”。
3. 把 `tools / resources / prompts` 和 `roots / sampling` 一起看，才能真正理解 MCP 的双边职责，而不是把它误解成单向的 server 能力暴露协议。

## relation to Agent engineering

这篇内容和你后面做 Agent 工程的关系很直接。

第一层是工作区安全边界。你以后做代码 Agent、文档 Agent、运维排障 Agent，只要涉及文件系统，就应该先想“哪些目录是显式授权的 roots”。不要靠当前进程工作目录、相对路径或实现习惯去隐式决定权限面。

第二层是模型调用治理。如果以后你做 MCP server，不要默认它自己应该直接持有模型凭证。很多企业场景更合理的方式，是由 host / client 统一持有模型接入和预算控制，再通过 `sampling` 给 server 提供受控的推理能力。

第三层是排障思路。以后如果一个 Agent 表现异常，不要只问“tool 调错了吗”。也可能是 roots 提供错了，导致上下文边界不对；也可能是 sampling 被用户拒绝、模型映射不符合预期、或者 host 刻意裁剪了上下文。

第四层是和 Java / Spring 经验桥接。你熟悉的网关、权限中心、资源白名单、统一审计，在 MCP/Agent 世界里并没有消失，只是换成了 `roots`、`sampling`、host control、capability negotiation 这些新名字。

## a small action for tonight

今晚花 30 分钟做一个“代码仓库 Agent 边界草图”：

1. 选一个你熟悉的场景，比如“巡检脚本助手”或“代码改造助手”。
2. 先列出它应该只拿到的 3 个 `roots`，例如主仓库、配置仓库、日志导出目录。
3. 再单独写 5 行规则，规定哪些情况下 server 可以发起 `sampling`，哪些情况下必须用户确认，哪些输出不能直接回传给 server。
4. 最后补一句话：如果 `roots` 变更或 `sampling` 被拒绝，这个 Agent 应该怎样降级，而不是直接报一个模糊错误。

你今晚如果能把这两层写清楚，MCP 在你脑子里就不再只是“工具协议”，而会开始像一个真正的 Agent 平台边界模型。

## 原文关键段落翻译（人工翻译，放在文末）

1. `roots` 规范把根目录定义为客户端暴露给服务器的文件系统边界，用来告诉服务器哪些目录和文件可访问，并支持在边界变化时通知服务器。
2. 支持 `roots` 的客户端必须声明 `roots` capability；服务器通过 `roots/list` 获取 roots，若客户端支持 `listChanged`，则在 roots 变更时发送 `notifications/roots/list_changed`。
3. `roots` 的安全要求包括：客户端只暴露具备适当权限的 roots、校验 root URI 防止路径穿越；服务器则应尊重根目录边界并对操作路径做校验。
4. `sampling` 规范允许服务器通过客户端请求一次模型生成，从而让客户端保留对模型访问、模型选择和权限控制的主导权，甚至不需要 server 自己持有模型 API key。
5. 客户端在开始采样前应让用户检查并批准请求，在把采样结果返回给服务器前也应允许用户检查结果，这体现了 human in the loop 的设计。
6. `sampling` 里的 `modelPreferences` 和 `hints` 只是建议，最终选哪个模型由客户端决定；客户端也可以把 server 的模型提示映射为自己可用的等价模型。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://modelcontextprotocol.io/specification/2025-06-18/client/roots
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://modelcontextprotocol.io/specification/2025-06-18/client/sampling
