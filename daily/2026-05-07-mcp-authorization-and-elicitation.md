# title

2026-05-07 为什么做 MCP 集成时，不能只会配 `tools`，还必须补上 `authorization` 和 `elicitation`

## original source

- 标题：Authorization | Model Context Protocol Specification 2025-06-18
- 链接：https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization
- 来源类型：官方规范
- 规范版本：2025-06-18
- 访问时间：2026-05-07

- 标题：Elicitation | Model Context Protocol Specification 2025-06-18
- 链接：https://modelcontextprotocol.io/specification/2025-06-18/client/elicitation
- 来源类型：官方规范
- 规范版本：2025-06-18
- 访问时间：2026-05-07

## why read today

你这几天已经连续补了 `tools`、`resources`、`prompts`、`roots`、`sampling`。如果今天继续只看“server 暴露了什么能力”，理解还是会缺一块最像企业交付现场的东西：安全边界和人工交互边界。

对你这种做过 10 年 Java / Spring / IoT 集成 / ToB 交付的人来说，真正决定系统能不能落地的，往往不是“有没有接口”，而是另外三件事：

1. 凭证到底归谁持有，谁负责发 token，谁负责校验 token。
2. 出现 401、403、权限不足时，客户端和服务端各自该怎么处理。
3. 当 server 需要向用户补资料时，谁来弹窗、谁来解释、谁来保留拒绝权。

`authorization` 讲的是第一类和第二类问题，`elicitation` 讲的是第三类问题。把这两篇补上，你对 MCP 的理解才会从“接口协议”升级到“可治理的 Agent 平台协议”。

## original-text translation

`authorization` 这篇规范首先把 MCP 的授权层级说得很明确：它处理的是传输层上的授权问题，让 MCP client 能代表资源所有者去访问受保护的 MCP server。对于基于 HTTP 的 MCP 连接，如果实现了授权，就应该遵循这套规范；如果是 STDIO 传输，则不应该照搬这套 OAuth 流程，而是更适合直接从环境里取凭证。

规范把 MCP 里的角色对应到了 OAuth 世界：受保护的 MCP server 相当于 OAuth 资源服务器，MCP client 相当于 OAuth 客户端，授权服务器负责在需要时和用户交互并发放 access token。MCP server 必须通过受保护资源元数据告诉 client 自己对应哪些授权服务器；当返回 `401 Unauthorized` 时，还必须用 `WWW-Authenticate` 头把相关元数据位置指给 client。

它还强调了几个很工程化的要求。第一，client 在发起授权请求和 token 请求时，都必须带上 `resource` 参数，明确这次 token 是给哪个 MCP server 用的。第二，client 对 server 的每一个 HTTP 请求都必须带 `Authorization: Bearer <access-token>` 头，而不是只在“建连时授权一次”。第三，server 必须校验 token 的 audience，确认这个 token 就是发给自己用的，而不是别的系统的 token 顺手拿来复用。

`elicitation` 这篇规范讲的是另一件事：当 server 在执行过程中还缺一些信息时，它可以通过 client 去向用户请求补充信息。规范把这个能力定义成一种标准化的人机交互请求，server 发 `elicitation/create`，并通过一个受限的 JSON Schema 描述自己想收集的数据结构。

这里最重要的不是“server 可以继续问用户问题”，而是“client 仍然掌握用户交互的控制权”。规范明确说，server 不能用 elicitation 去索要敏感信息；应用应当让用户看清楚是哪个 server 在请求信息，应允许用户修改后再提交，也要给出清晰的 `decline` 和 `cancel` 选项。返回结果不是简单的成功失败，而是三种动作：`accept`、`decline`、`cancel`，让 server 能区分“明确拒绝”和“用户只是先关闭了交互框”。

## Chinese deep summary

如果把今天这两篇规范压成一句工程判断，就是：MCP 不只是让 server 把能力挂出来，还规定了“client 如何代表用户拿权限”和“server 如何通过 client 向用户拿补充信息”。这两层如果不补，Agent 系统就很容易停留在 Demo 阶段。

先看 `authorization`。

很多人第一次接 MCP，会把它理解成“模型前面的一个工具总线”。这个理解不算错，但太薄。只要你开始接真实企业系统，就会立刻碰到你过去非常熟悉的问题：凭证放哪，授权链路走哪，失败后是重试、重新登录还是换权限，系统 A 的 token 能不能拿去打系统 B。MCP 的 `authorization` 规范本质上是在说：这些问题不能继续藏在“各家自己搞”的实现细节里，而应该被明确纳入协议约束。

这里最值得你注意的是 `resource` 参数和 audience 校验。它们背后表达的是一个非常 ToB 的原则：token 不是“拿到了就到处能用”，而是应该绑定到明确的目标资源。你以前做 IoT 平台集成、企业网关、内部服务授权时，一定见过“凭证复用过头导致越权”的问题。MCP 在这里吸收的是同一套教训。client 不但要去拿 token，还必须清楚声明“我要访问的是哪一个 MCP server”；server 也不能只验签名和过期时间，还要确认这个 token 的受众确实是自己。否则就会出现典型的 confused deputy 问题：一个本来只该访问 A 的 token，被 B 也收下了。

再往下看，`authorization` 还顺手纠正了一个很容易在 Agent 场景里犯的误区：不是所有 MCP 接入都应该照着浏览器 OAuth 流程来。规范明确说，HTTP 传输适合这套授权规范，STDIO 传输则不应该照搬，而更适合从环境里取凭证。这个区分很重要，因为它提醒你不要为了“统一”而机械套协议。你以后自己做本地 Agent 工具链时，很可能同时面对这两类场景：本地 CLI/IDE 内嵌的 MCP server，更像进程内或同机协作；远端 SaaS MCP server，则明显需要标准 OAuth 边界。两边的安全设计不该一样。

再看 `elicitation`。

如果说 `sampling` 解决的是“server 想借 client 的模型能力做一次生成”，那 `elicitation` 解决的就是“server 想借 client 的用户交互能力向人再问一次”。这对 Agent 工程非常关键，因为现实世界里的任务并不总能靠现有上下文一次做完。很多时候 server 缺的是用户名、环境选择、部署窗口、确认范围、回滚策略、审批意见。这类信息如果让 server 自己随便拼 prompt 去“问模型猜”，那就不叫可靠系统；如果让 server 自己直接控制 UI，又会把宿主应用的安全和一致性交出去。

`elicitation` 的设计刚好卡在中间：server 可以表达需要什么信息，但不能越过 client 直接控制用户；client 负责把这个请求呈现给用户，并保留拒绝、取消、修改后提交的权利。你可以把它理解为 MCP 里的“标准化补单接口”，只是这里补的不是业务单据，而是运行时缺失的人类输入。

这对你的背景特别有帮助，因为你本来就熟悉“交付系统最后一公里”的问题。很多 Java 后台工程师一转 Agent，会自然把重点全放在 prompt、tool schema、workflow graph 上；但真正上线以后，用户交互补资料、权限审批、异常分支、人工接管才是稳定性来源。`elicitation` 规范之所以重要，不是因为它酷，而是因为它把这些“以前总靠前端临时写弹窗”的东西，提升成了协议层的统一接口。

另外，`elicitation` 限制 schema 只能是扁平对象、基础类型和枚举，这一点也很值得你记。它不是功能弱，而是有意压复杂度，好让不同 client 更容易稳定实现。你可以把它类比成企业集成里的“受控表单输入协议”：宁可限制一点表达能力，也要提高跨宿主实现的一致性和可验证性。对 Agent 平台来说，这比支持一个无限复杂但各家都实现不一致的动态表单系统更有工程价值。

把今天这两篇和前几天的 `tools / resources / prompts / roots / sampling` 合起来看，MCP 的轮廓会清楚很多：

1. `tools / resources / prompts` 更偏 server 暴露什么。
2. `roots / authorization / elicitation / sampling` 更偏 client 和 host 保留什么控制权。

这正是企业级 Agent 平台和玩具 Demo 的分水岭。Demo 只关心“模型能不能调起来”；平台必须关心“谁授权、谁审批、谁能拒绝、谁来审计、错了怎么降级”。

## 3 key takeaways

1. `authorization` 的重点不是“接一下 OAuth”这么简单，而是把 token 获取、目标资源绑定、401/403 处理、audience 校验这些企业级安全边界协议化。
2. `elicitation` 的重点不是“让 server 能多问一句”，而是把运行时向用户补资料这件事交还给 client 管理，并保留 `accept / decline / cancel` 三种明确分支。
3. MCP 真正成熟的地方，在于它不仅定义 server 能力暴露，还定义 host/client 如何保留授权、人机交互和最终放行控制权。

## relation to Agent engineering

这篇内容和你后面做 Agent 工程的关系非常直接。

第一层，是你做远端 MCP 集成时的安全设计。以后不管你是把 Agent 接到公司内网知识库、工单系统、设备平台，还是某个第三方 SaaS MCP server，都不应该把“拿个 token 打过去”当成完成接入。你应该明确设计：

1. token 是 host 统一持有，还是某个独立 client 持有。
2. `resource` 参数怎么生成，canonical URI 怎么定。
3. 401 和 403 分别触发什么恢复动作。
4. server 如何验证 audience，避免 token passthrough。

第二层，是你做多步骤 Agent 工作流时的补信息机制。以后你做“代码改造助手”“运维执行助手”“客户环境巡检助手”，都很容易遇到信息缺口。与其在 prompt 里模糊地让模型“如果缺信息就继续问”，不如明确区分：

1. 可以由模型自己推断的上下文。
2. 必须通过 `elicitation` 向用户确认的输入。
3. 明确拒绝后应该怎样降级。
4. 用户取消后是否可以稍后重试。

第三层，是你从 Java/Spring 后台走向 Agent 平台时最该建立的新习惯：不要把宿主应用看成一个透明壳。它不是只负责“把 server 接上去”，而是负责统一治理模型访问、凭证流转、用户审批和交互入口。这和你过去做网关、权限中心、集成平台时的控制面思维，其实是一脉相承的。

## a small action for tonight

今晚花 30 分钟，写一个你自己的《MCP 集成最小治理清单》，只写一页：

1. 选一个你最可能先做的 Agent，例如“代码库改造助手”或“客户环境排障助手”。
2. 写 5 行，定义它接远端 MCP server 时的授权规则：token 从哪来，`resource` 怎么定，401/403 怎么处理。
3. 再写 5 行，定义它什么时候允许触发 `elicitation`，哪些问题绝不能问，用户 `decline` 和 `cancel` 时分别怎么降级。
4. 最后补一句：如果明天让你用 Spring Boot 做这个 host 层，你会把这些规则放在哪个模块里。

## 原文关键段落翻译（人工翻译，放在文末）

1. MCP 在传输层提供授权能力，让 client 可以代表资源所有者访问受保护的 MCP server；这套规范主要定义的是基于 HTTP 传输的授权流程。
2. MCP server 必须实现受保护资源元数据，并在返回 `401 Unauthorized` 时通过 `WWW-Authenticate` 头告诉 client 去哪里发现授权服务器信息。
3. MCP client 在授权请求和 token 请求里都必须带上 `resource` 参数，明确这次 token 是为目标 MCP server 申请的。
4. MCP server 必须校验 access token 的 audience，确认 token 是专门发给自己使用的；无效或过期 token 必须返回 `401`。
5. Elicitation 让 server 可以通过 client 在交互过程中向用户请求额外信息，同时保持 client 对用户交互和数据共享的控制权。
6. Server 不能用 elicitation 索要敏感信息；client 应让用户看清楚是谁在请求信息，并允许用户在发送前检查、修改、拒绝或取消。
7. Elicitation 的返回动作必须明确区分 `accept`、`decline` 和 `cancel`，这样 server 才能把“明确拒绝”和“暂时关闭”当成不同分支处理。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://modelcontextprotocol.io/specification/2025-06-18/client/elicitation
