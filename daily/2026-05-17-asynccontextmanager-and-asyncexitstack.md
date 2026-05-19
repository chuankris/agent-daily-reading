# title

2026-05-17 为什么 `asynccontextmanager` 和 `AsyncExitStack` 是你写稳 Agent 宿主的资源边界课

## original source

- 标题：`contextlib` — Utilities for `with`-statement contexts
- 链接：https://docs.python.org/3/library/contextlib.html
- 来源类型：Python 官方文档
- 访问时间：2026-05-17

- 标题：3. Data model — With Statement Context Managers / Asynchronous Context Managers
- 链接：https://docs.python.org/3/reference/datamodel.html#asynchronous-context-managers
- 来源类型：Python 官方语言参考
- 访问时间：2026-05-17

## why read today

你前几天已经补了 `TaskGroup`、取消语义、`contextvars`、异常组、类型契约。现在还差一个经常被初学者忽略、但在 Agent host 里非常致命的层：资源生命周期。

对你这种做过很多年 Java / Spring / IoT 集成 / ToB 交付的人，这个点其实不陌生。你以前很自然会关心连接池、事务边界、超时回收、日志上下文、失败清理、回调释放。到了 Python Agent 工程里，这些问题没有消失，只是从 `try/finally`、拦截器、容器生命周期，换成了 `async with`、`asynccontextmanager`、`AsyncExitStack` 这一套表达方式。

今天这篇的价值不是学一个“语法糖”，而是补齐一个真实 Agent 运行时每天都在做的事情：打开模型客户端、注册 trace、申请临时工作目录、持有 websocket/http 会话、收集工具资源、在任何中途失败时把它们按正确顺序释放。

## original-text translation

Python 官方语言参考先给了上下文管理器最底层的定义：上下文管理器是一个对象，它定义了执行某段代码时需要建立的运行时上下文，并负责在代码块进入时做准备、在退出时做清理。同步版依靠 `__enter__` / `__exit__`，异步版依靠 `__aenter__` / `__aexit__`，区别只是异步版本允许在进入和退出阶段挂起等待。

`contextlib` 文档则把工程层的用法补齐了。`asynccontextmanager` 允许你用一个“异步生成器 + `try/finally`”来定义 `async with` 可用的上下文管理器，而不必手写一个类；它表达的是“拿资源”和“放资源”必须成对出现。`AsyncExitStack` 则进一步解决了“资源数量不是写死的，而且有些是同步清理、有些是异步清理”的问题。它本质上维护一个后进先出的退出栈，你在运行过程中逐个注册清理逻辑，离开时由它统一逆序执行。官方示例特别强调：即使后面某一步打开资源失败，前面已经成功拿到的资源也会被自动释放。

换句话说，这两份官方文档想传达的不是“with 语句更优雅”，而是：资源边界要成为语言级、结构化、可组合的工程契约。

## Chinese deep summary

如果把今天这组材料压缩成一句话，那就是：`asynccontextmanager` 负责把“单个资源的获取/释放”写清楚，`AsyncExitStack` 负责把“一批动态资源的统一收口”写清楚。

为什么这对 Agent 工程特别关键？因为 Agent 宿主最怕的不是单次调用失败，而是“做到一半失败，前面借来的资源没还干净”。比如一次请求里，你可能会：

1. 建一个模型调用会话。
2. 打开一个或多个 MCP client 连接。
3. 注册 tracing span 或事件流。
4. 创建临时文件、缓存目录、回放工件。
5. 申请数据库、向量库、消息队列或浏览器会话。

这些资源的共同特点是：它们经常不是在函数开头一次性声明好的，而是随着执行过程动态拿到的。你先决定要不要检索，再决定要不要开工具，再决定要不要启观察或持久化。于是，普通的多层 `try/finally` 很快就会变得难维护，释放顺序也容易乱。

`asynccontextmanager` 先解决“局部资源边界”的问题。它鼓励你把一个资源的进入和退出逻辑写在一起，而不是散落在主流程各处。这个习惯对你尤其重要，因为你原来在 Spring 里很多生命周期是容器帮你兜底的；换到 Python 小型 Agent 宿主后，如果你不主动把边界显式化，代码很容易演化成“主流程全是业务，清理逻辑全靠记忆补”。

`AsyncExitStack` 则解决“组合资源边界”的问题。它适合这样一种典型场景：你事先并不知道最终会拿到多少资源，也不知道它们全部是同步还是异步的，但你知道只要成功拿到一个，就必须在最后安全释放。这个模型和真实 Agent host 非常贴近，因为工具、连接、回调、临时对象往往都是条件性出现的。

这里最值得你建立的工程直觉有三个。

第一，资源释放顺序不是细节，而是契约。`AsyncExitStack` 按后进先出逆序清理，本质上是在维护依赖顺序。后拿到的资源往往依赖前面的基础资源，所以必须先释放更内层、再释放更外层。这和你以前做集成系统时“先关业务句柄，再关底层连接”的判断是同一件事。

第二，清理逻辑不应该依赖“主流程顺利走完”。官方示例反复强调，中途失败也会自动回收前面成功进入的上下文。这对 Agent 系统非常重要，因为模型超时、工具异常、网络断连、人工中断都属于常态，不是边缘情况。

第三，资源管理代码值得独立抽象。很多 Python 新手会把客户端创建、日志 span、临时目录、回调注册都直接写在业务函数里，后面再用很多散乱的 `await close()` 修补。更稳的做法是：把每一种资源封装成上下文管理器或注册到 `AsyncExitStack`，让主流程只表达编排，不负责记住每一个清理细节。

如果用你熟悉的 Java 经验类比，可以这样理解：

1. `asynccontextmanager` 很像你手写一个明确成对的 acquire/release 封装，只是 Python 用 `yield` 把边界写进语言结构里。
2. `AsyncExitStack` 很像“动态版的组合 try-with-resources”，而且能混合同步与异步清理。
3. `async with` 在 Agent host 中承担的角色，接近“轻量容器生命周期边界”，只是粒度更贴近一次请求或一次运行。

这篇材料还会直接影响你后面如何组织代码。一个健康的 Agent 宿主，通常会把“编排逻辑”和“资源生命周期逻辑”分开：主流程负责决定先检索还是先调工具，资源层负责保证无论哪一步出错，连接、trace、回调、临时工件都能正确回收。今天这篇实际上是在给你后面的 Python 工程化打地基。

## 3 key takeaways

1. `asynccontextmanager` 适合定义单个异步资源的获取与释放边界，让进入逻辑和退出逻辑天然成对。
2. `AsyncExitStack` 适合处理运行过程中动态出现的多种资源，并保证它们按逆序统一清理，即使中途失败也不会漏收口。
3. 对 Agent 工程来说，资源生命周期不是附属细节，而是宿主稳定性的核心契约；连接、trace、临时文件、工具会话都应该被结构化管理。

## relation to Agent engineering

这篇内容和 Agent 工程的关系非常直接。

第一层关系是 Agent host 的稳定性。只要你开始写真正可跑的宿主，而不是 notebook 级 demo，就一定会遇到“资源先拿了一半，后面失败了怎么办”。`asynccontextmanager` 和 `AsyncExitStack` 就是处理这类问题的基础件。

第二层关系是 MCP / tool 连接管理。一个请求可能按条件打开不同工具客户端，有的需要 `async with`，有的只需要注册关闭回调。`AsyncExitStack` 很适合把这些异构资源统一纳入一个退出栈。

第三层关系是 trace / eval / observability。评测、日志埋点、span、artifact 收集通常都需要“开始时注册，结束时无论成败都 flush/close”。如果你没有结构化资源边界，这些附属能力最容易在异常路径上失效。

第四层关系是你的迁移路径。你不缺系统集成的工程判断，缺的是把这套判断翻译成 Python 的宿主写法。今天这组官方资料，本质上就是把你熟悉的生命周期管理思维映射到 Python Agent 代码里。

## a small action for tonight

今晚做一个 30 分钟的小实验，只练资源收口，不追求业务复杂度：

1. 用 `@asynccontextmanager` 写一个假的 `get_model_client()`，进入时打印 `open model client`，退出时打印 `close model client`。
2. 再写一个假的 `get_trace_span()` 和一个临时目录清理函数，分别注册到 `AsyncExitStack`。
3. 在主流程里故意让第二步抛异常，观察已经进入的资源是否按逆序释放。
4. 把这个实验总结成一句自己的规范：以后凡是“需要关闭/flush/回收”的对象，优先做成上下文管理器或纳入退出栈，而不是散落在业务函数末尾手工关。

## 原文关键段落翻译（人工翻译，放在文末）

1. 上下文管理器定义的是执行某段代码时要建立的运行时上下文，并负责在进入时准备、在退出时清理。
2. 异步上下文管理器与同步版本语义相同，只是 `__aenter__()` 和 `__aexit__()` 必须返回可等待对象，因此进入和退出阶段可以执行异步操作。
3. `asynccontextmanager` 让你不必专门写一个类，就能把异步资源的获取和释放封装为 `async with` 可用的上下文管理器。
4. `AsyncExitStack` 适合把多个上下文管理器和清理回调组合起来统一管理；即使后续某一步进入失败，之前已成功进入的资源也会在退出时自动释放。
5. 退出栈按后进先出的顺序执行清理逻辑，这让资源依赖关系可以被安全地逆序拆除。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.python.org/3/library/contextlib.html
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.python.org/3/reference/datamodel.html%23asynchronous-context-managers
