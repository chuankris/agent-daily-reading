# title

2026-06-26 为什么现在该补 `MCP Tool Annotations`：别把工具描述当真相，要把它当成 Agent 风险边界的提示层

## original source

- 标题：Tools
- 链接：https://modelcontextprotocol.io/specification/2025-11-25/server/tools
- 来源类型：MCP 官方规范
- 访问时间：2026-06-26

- 标题：Tool Annotations as Risk Vocabulary: What Hints Can and Can't Do
- 链接：https://blog.modelcontextprotocol.io/posts/2026-03-16-tool-annotations/
- 来源类型：MCP 官方博客 / 一线工程经验总结
- 访问时间：2026-06-26

## why read today

你前面已经读过 MCP 的 tools contract、resources、authorization、Python SDK server，也知道一个 Agent 能“发现工具、调用工具、拿回结果”。但如果今天就让你做一个真正能接客户内网、知识库、工单系统、Shell、HTTP API 的 Agent，你马上会遇到一个更工程化的问题：

**客户端到底该怎样判断一个工具大概危险不危险，什么时候该自动放行，什么时候该二次确认，什么时候该直接拦？**

这正是 `tool annotations` 的位置。它不是协议里最显眼的部分，却非常像你以前做 Java / IoT / ToB 集成时那些“接口语义标签”：

- 这个接口是不是只读？
- 这次操作是不是破坏性的？
- 失败重试会不会出事故？
- 这个连接出去之后会不会把不可信内容带回来？

区别在于，MCP 这里面对的不是传统人写代码后再调接口，而是 **模型会自己选工具**。因此，annotations 的价值不是“文档更好看”，而是给 host / client 一个最基础的风险词汇表，让 Agent 运行时能做更稳的确认、提示、策略控制。

对你这种做过多年集成和交付的人来说，这篇最值钱的点也不是概念新，而是它把一个老问题换了新外形：

**接口元数据什么时候只是提示，什么时候可以参与策略，什么时候绝不能被当作强保证？**

这恰好是你从“能跑通工具调用”进阶到“能设计 Agent 安全边界”的关键一步。

## original-text translation

MCP 官方 `Tools` 规范先把工具的基本模型说清楚：server 可以暴露可被语言模型调用的 tools；客户端通过 `tools/list` 发现工具，通过 `tools/call` 调用工具。每个 tool 至少有 `name`、`description`、`inputSchema`，还可以有 `outputSchema`、`annotations`、`execution` 等元数据。规范还特别强调一条安全前提：**除非这些 annotations 来自可信 server，否则客户端必须把它们当作不可信信息。**

同一份规范对“人类参与”也给出明确方向。虽然 tools 是为模型自动发现和调用设计的，但协议建议应用始终保留 human in the loop，至少要让用户知道暴露了哪些工具、什么时候正在调工具，并在敏感操作前给确认入口。也就是说，tool annotations 的语境从一开始就不是“模型完全自由”，而是“模型可控调用 + 客户端负责风险交互”。

MCP 官方博客进一步把 `ToolAnnotations` 当前这层语义讲透了。现在的接口里，常见提示包括：

- `readOnlyHint`：这个工具是否只读，不修改环境；
- `destructiveHint`：如果会改，是否是破坏性修改；
- `idempotentHint`：同样参数重复调用是否安全；
- `openWorldHint`：这个工具是否会接触开放外部世界，而不是只处理一个封闭域。

博客反复强调：这些字段全部都只是 hint，不是 contract。默认姿态也故意偏保守：如果一个工具没写 annotations，就应默认把它看成“不是只读、可能破坏、不可安全重试、而且连着开放世界”。这很像交付现场里你还没拿到明确接口说明前，先按最坏情况设计保护带。

博客接着解释了 annotations 真正能做什么。它们可以帮助客户端决定是否弹确认框，比如可信 server 上的 `readOnlyHint: true` 可能允许自动放行，而 `destructiveHint: true` 则更适合先提醒用户；也可以作为策略引擎输入，例如“访问过私有数据后，不允许开放世界工具直接外发结果”。但博客同样说得很死：annotations **不能**让模型自动免疫 prompt injection，**不能**阻止恶意 server 说谎，**不能**代替 sandbox、网络控制、权限隔离这类硬约束。

最有工程味的一段，是博客把风险从“单个工具属性”提升到“会话组合风险”。文中借用了 Simon Willison 提出的 lethal trifecta：只要一个会话同时具备“可读私有数据、可接触不可信内容、可对外通信”这三种能力，就可能形成数据外泄链。换句话说，`search_emails` 单看也许不危险，但一旦和浏览器抓取、外发 HTTP、Shell 执行放到同一个 session，风险就变了。annotations 的价值之一，就是帮助 client 看见这种组合，而不是只盯着某一个工具。

最后，博客给了一个很实用的判断框架：如果一个新 annotation 不能触发明确的 client 行为变化，它就不该轻易进规范；如果它的价值完全依赖“这个字段必须是真的”，那它更像 contract，不该只放在 hint 层。对正在补 Agent 工程基础的人来说，这个区分特别关键：**metadata 可以帮助调度与提醒，但真正兜底的安全保证，必须落在 deterministic control 上。**

## Chinese deep summary

如果把今天这两份材料压成一句工程判断，就是：

**MCP 的 tool annotations 不是“给工具打标签的小装饰”，而是 Agent host 用来组织确认、权限和风险感知的第一层语义外壳。**

第一层，你要先把 annotations 和“接口事实”分开。

很多刚接触 Agent 的人一看到 `readOnlyHint`、`destructiveHint` 这种字段，会下意识把它们理解成“这个工具的真实属性”。但规范和官方博客都在提醒你，这种理解是危险的。因为这些字段来自 server 自报，server 可能写错，也可能故意说谎。  
所以更准确的理解应该是：

- 它是 server 给 client 的行为提示；
- 它可以参与 UI、确认、策略判断；
- 它不能直接当作强安全保证。

这和你以前做系统集成时很像。客户接口文档上写“查询接口，无副作用”，你会拿它做设计参考，但不会因此取消审计、权限和回滚预案。MCP 这里只是把这件事制度化了。

第二层，annotations 真正补的是 host 侧，而不是 model 侧。

这点特别重要。很多人会把安全问题都甩给模型本身，比如“模型应该识别恶意提示”“模型不该被 prompt injection 诱导”。但官方博客讲得很清楚：tool annotations 不负责让模型更聪明，它负责让 host 更清醒。

也就是说，它主要服务这些动作：

- 调用前要不要确认；
- 失败后能不能自动重试；
- 这个工具输出是否可能带回不可信内容；
- 当前 session 是否已经跨过了某条信任边界。

从工程视角看，这其实很符合你熟悉的架构分层。模型负责理解和规划，host 负责控制和兜底；不要把本该在运行时治理层做的事情，寄希望于模型“自己懂分寸”。

第三层，`openWorldHint` 是最值得你额外重视的那个字段。

`readOnlyHint`、`destructiveHint`、`idempotentHint` 都比较像传统后端工程里的语义标签，容易理解；但 `openWorldHint` 指向的是另一类问题：**这个工具是否把 session 接到了不可预测的外部世界。**

这对 Agent 比传统业务系统更关键，因为一旦工具能读网页、邮件、日历、搜索结果、外部 API 返回值，模型就可能把这些内容也当作可执行指令背景。对你这个长期做 ToB 集成的人来说，可以把它类比成：

- 内网主数据表：封闭域；
- 客户外部邮件正文：开放域；
- 本地配置文件读取：相对封闭；
- 浏览器抓到的页面内容：开放域。

只要开放域内容和“可读私有数据”“可外发能力”在同一个 session 里相遇，风险就不再是单点风险，而是链路风险。

第四层，你应该把 tool annotations 理解成 Agent 时代的“前置风险词汇表”。

为什么说是词汇表？因为它首先解决的是“怎么描述”，然后才是“怎么治理”。没有这些词汇，host 很难统一表达“这类工具需要确认”“这类工具可以重试”“这类工具跨越了信任边界”。有了这些词汇，才可能继续往上做：

- approval policy
- session taint 标记
- tool allowlist / denylist
- 多工具组合的风险检测
- 更细的审计和可观测性

这和你以前做企业集成平台一样。先统一接口语义，后面才能统一熔断、鉴权、重试、审计。annotations 在 MCP 里扮演的，就是这个“先统一语义再统一治理”的入口角色。

第五层，默认按最坏情况处理，是一个很适合你当前阶段建立的习惯。

官方博客提到，没写 annotations 的工具，默认应当被视为非只读、可能破坏、不可安全重试、且处于开放世界。这一点非常值得你拿来形成工程直觉。因为转型初期最容易犯的错误，就是看到 demo 能跑，就先假设工具“问题不大”。  
但做交付的人应该反过来：

- 没写明只读，就按可能写入处理；
- 没写明幂等，就不要随便自动重试；
- 没写明封闭域，就按可能带回不可信内容处理；
- 没有硬约束，就不要当作安全保证。

这套保守心智，会直接影响你以后设计 Agent runtime、审批流、沙箱、审计链路时的质量。

第六层，从你的背景出发，今天这篇最重要的迁移不是“记住四个 hint 名字”，而是把它映射到你熟悉的工程治理经验。

你以前在 Java / Spring / IoT 集成里做过很多这样的事情：

- 区分查询接口和写接口；
- 区分可重试操作和不可重试操作；
- 区分内网封闭系统和公网开放系统；
- 把元数据、权限、确认、审计串成一条治理链。

现在做 Agent，本质上还是这些事情，只是对象从“接口调用者”变成了“模型 + host + tool session”。  
当你能把 MCP annotations 看成这条治理链的起点，而不是一个 schema 细节，你就开始真正进入 Agent engineering 了。

## 3 key takeaways

1. `tool annotations` 是提示层，不是事实层，更不是强制约束层；来自不可信 server 的 annotations 不能直接当作安全保证。
2. annotations 的主要价值在 host / client：驱动确认、重试策略、风险提示和会话级策略，而不是让模型自动抵抗 prompt injection。
3. 真正危险的往往不是单个工具，而是多工具组合；尤其当“私有数据访问 + 不可信内容输入 + 对外通信”同时存在时，session 风险会陡增。

## relation to Agent engineering

这篇内容和 Agent engineering 的关系非常直接，因为它补的是“工具可用”之后的下一层：**工具治理**。

第一，它让你意识到，Agent 工程里真正难的不是把工具挂上去，而是决定：

- 哪些工具可以默认调用；
- 哪些工具必须人工确认；
- 哪些工具一旦进入 session，就要提高后续所有调用的防护等级。

第二，它把你之前读过的几条线串起来了：

- `tools/list` / `tools/call` 是发现和调用面；
- `structured outputs` / schema 是契约面；
- `annotations` 是风险语义面；
- `authorization`、sandbox、network policy 才是强控制面。

这四层不能混。混了以后，你会不是高估 metadata，就是低估运行时治理。

第三，它对你的背景特别友好。因为你原来最擅长的并不是“模型技巧”，而是把边界复杂、参与方多、风险不均匀的系统接起来并交付稳定。tool annotations 正好是 Agent 世界里“边界语义治理”的一个很好入口。

## a small action for tonight

今晚做一个 30 到 40 分钟的小练习，不求写很多代码，只求把风险意识落地：

1. 随便找一个你已经看过的 MCP server 例子，列出它暴露的所有 tools。
2. 手工给每个 tool 补一个表格：`是否只读`、`是否破坏性`、`是否可安全重试`、`是否接触开放世界`。
3. 再额外补两列：`如果 server 不可信，我还敢相信这一列吗？`、`真正兜底应该靠什么硬控制？`
4. 最后把这些 tool 按 session 组合一遍，专门找“能读私有数据 + 能接触开放内容 + 能外发”的组合。
5. 用一句话写结论：`我的风险判断不能只看单个工具定义，还要看整个会话能力拼装。`

## 原文关键段落翻译（人工翻译，放在文末）

1. MCP 允许 server 暴露可被语言模型调用的工具；这些工具让模型能够与外部系统交互，例如查询数据库、调用 API 或执行计算。每个工具都通过名称唯一标识，并带有描述其 schema 的元数据。
2. 工具在 MCP 中被设计为由模型控制，模型可以基于上下文理解和用户提示自动发现并调用工具；但从信任、安全和保障角度，应用始终应保留人类在环，并为敏感操作提供确认能力。
3. 工具定义可以包含 `annotations` 这一组可选属性，用于描述工具行为；但客户端必须把来自不可信 server 的 annotations 视为不可信。
4. 当前 `ToolAnnotations` 里的字段本质上都是 hint。它们不是对工具行为的保证；如果没有 annotations，最安全的姿态是按最坏情况处理。
5. annotations 可以驱动确认提示、渐进式信任、用户体验优化和策略引擎输入；但它们不能让模型抵抗 prompt injection，也不能代替 sandbox、网络控制或其他硬约束。
6. 一个工具本身是否危险，往往取决于它和其他工具一起出现在什么 session 里；真正该警惕的是会话级的能力组合，而不只是单个工具描述。
7. 如果一个 annotation 不能触发明确的 client 行为变化，它就不适合进入协议；如果它的价值依赖“它必须是真的”，那它更像 contract，而不是 hint。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://modelcontextprotocol.io/specification/2025-11-25/server/tools
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://blog.modelcontextprotocol.io/posts/2026-03-16-tool-annotations/
