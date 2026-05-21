# title

2026-05-21 为什么 `Semaphore`、`as_completed()` 和 `to_thread()` 是你把 Agent 工具调用做成“可控并发”而不是“同时乱跑”的关键一课

## original source

- 标题：Synchronization Primitives — Python 3.14.5 documentation
- 链接：https://docs.python.org/3/library/asyncio-sync.html
- 来源类型：Python 官方文档
- 访问时间：2026-05-21

- 标题：Coroutines and Tasks — Python 3.14.5 documentation
- 链接：https://docs.python.org/3/library/asyncio-task.html
- 来源类型：Python 官方文档
- 访问时间：2026-05-21

## why read today

你前几天已经连续补了 `TaskGroup`、取消语义、异常组、`Queue` 背压、`httpx` 超时和重试、`OpenAPI/JSON Schema` 契约。现在还差一个很典型、也很容易在真实 Agent 系统里踩坑的工程问题：

1. 多个工具调用能并发，但不能无限并发，怎么收口？
2. 多个子任务同时跑时，结果要按“谁先完成谁先处理”，而不是等最慢那个拖全局，怎么写？
3. 现实里总会遇到阻塞型 SDK、旧版数据库驱动、文件系统操作、客户侧私有库，它们不是异步的，怎么安全接进 `asyncio` 宿主？

这正好卡在你的当前迁移点上。你过去做 Java / Spring / IoT 集成时，一定很熟悉线程池大小、下游连接数、接口限流、先返回先处理、阻塞式第三方 SDK 包装这些事情。到了 Python Agent 工程里，这些判断没有消失，只是换成了 `asyncio.Semaphore`、`asyncio.BoundedSemaphore`、`asyncio.as_completed()`、`asyncio.to_thread()` 这套表达。

今天这篇值钱的地方，不是多记几个 API，而是把“Agent 宿主如何安全并发地调工具”这件事讲顺。

## original-text translation

Python 官方在 `asyncio` 同步原语文档里先讲了两条总规则：这些原语和 `threading` 的思路类似，但它们不是线程安全对象；同时，这些同步原语的方法本身不接受 `timeout` 参数，如果要做超时控制，应在外层配合 `asyncio.wait_for()` 之类的超时机制。

在 `Semaphore` 一节里，官方把核心语义讲得很直接：信号量维护一个内部计数器，每次 `acquire()` 时减一，每次 `release()` 时加一；计数器不能降到 0 以下，当 `acquire()` 发现计数器为 0 时，会阻塞等待，直到其他任务调用 `release()`。推荐的使用方式是 `async with sem:`，这样可以保证退出时正确释放。

官方还特别区分了 `Semaphore` 和 `BoundedSemaphore`：普通 `Semaphore` 允许 `release()` 调用次数多于 `acquire()`；而 `BoundedSemaphore` 在 `release()` 使内部计数超过初始值时会抛出 `ValueError`。这意味着后者更像一个“带自检的限流闸门”，可以帮助你尽早发现释放次数写错的并发 bug。

在 `Coroutines and Tasks` 文档里，`as_completed()` 的定位是：并发运行一组 awaitable，并在它们完成时按完成顺序逐个取得结果，而不是按提交顺序统一等待。Python 3.13 之后，它返回的对象既可以用作异步迭代器，也可以当作普通迭代器。文档还说明，如果设置了 `timeout`，而在超时前并不是所有 awaitable 都完成，就会抛出 `TimeoutError`。

同一页里的 `to_thread()` 则解决了另一个常见问题：把一个原本会阻塞事件循环的同步函数，放到单独线程里异步执行，并在等待结果时不阻塞事件循环。官方强调它主要用于 IO-bound 的阻塞函数；同时，当前的 `contextvars.Context` 会被传播到新线程，这意味着日志上下文、trace id 一类的上下文变量可以跟过去。

## Chinese deep summary

如果把今天这组材料压成一句话，那就是：`Semaphore` 决定“同一时间最多放多少活并发出去”，`as_completed()` 决定“谁先做完就先处理谁”，`to_thread()` 决定“遇到阻塞型遗留库时，别把整个事件循环一起拖死”。

这三件事为什么特别适合你现在读？因为你已经走过“概念理解”阶段，正在进入“宿主如何承载 Agent 工作流”的阶段。到这一步，最容易犯的错不是不会写 prompt，也不是不会接模型，而是把并发当成“开得越多越快”。

在真实 Agent 系统里，并发通常受三种上限约束：

1. 外部系统上限，比如搜索 API、数据库连接池、浏览器实例数、企业内部工具网关 QPS。
2. 本机资源上限，比如文件句柄、CPU、内存、socket 连接数。
3. 业务正确性上限，比如某些任务虽然能并发，但不允许同时改同一份状态。

`asyncio.Semaphore(n)` 解决的是第一类和第二类最常见的问题。它不是“让代码显得专业一点”的装饰，而是显式承认一个工程事实：你的 Agent 宿主吞吐能力是有限的。如果你把 50 个工具调用直接 `gather()` 出去，而下游供应商只允许 5 个并发连接，那你不是在提速，而是在把失败放大。

对你这种做过系统集成的人，这个概念并不陌生。你以前在 Java 里会配线程池大小、JDBC 连接池、MQ consumer 并发度、本地缓存预热线程数。`Semaphore` 在 Python 里扮演的角色非常接近“并发许可证池”。区别只是：在 `asyncio` 模型下，它保护的是协程级并发，而不是操作系统线程。

这里一个很容易被忽略的点，是普通 `Semaphore` 和 `BoundedSemaphore` 的差别。很多人觉得它们只是“严格一点”和“不严格一点”，但放到 Agent 工程里，它们的意义更像：

1. `Semaphore` 适合一般限流。
2. `BoundedSemaphore` 适合你特别怕写错释放逻辑、特别想尽早暴露 bug 的地方。

为什么？因为 Agent 工具调用链经常很长：鉴权、参数转换、远程调用、结果解析、落盘、trace、收尾。链路一长，就更容易因为异常分支、提前返回、超时取消而把 `release()` 配平写错。普通 `Semaphore` 即便多释放了，也不会立刻告诉你；而 `BoundedSemaphore` 会直接抛错，帮助你更早发现“许可证被释放过头”的契约错误。

再看 `as_completed()`。这是一个很适合 Agent 场景、但很多人没建立直觉的 API。很多初学者默认写法是 `await gather(*tasks)`，意思是“等大家都做完，再统一拿结果”。这在某些批处理场景没问题，但在 Agent 宿主里，常常会制造两个副作用：

1. 最慢的任务拖住最早完成的结果，增加整体尾延迟。
2. 你无法自然地做“谁先回来先消费、先显示、先写 trace、先触发下一步”。

`as_completed()` 的价值就是把“批量启动”和“逐个收割”拆开。比如你同时发起多个检索源、多个候选工具、多个健康检查，真正需要的往往不是“全部都完了再看”，而是“先完成的先进入下一阶段”。这很像你以前做系统集成时的并行探测、备用链路选择、多个接口抢先返回策略。

而且 Python 3.13 之后，这个 API 更顺手了：它既支持异步迭代，也能在异步迭代时保留原始 task/future 身份。这个细节对 Agent 工程很重要，因为你常常不仅关心“拿到什么结果”，还关心“这个结果是哪个工具、哪个 endpoint、哪个子任务回来的”。如果结果和原任务对象能自然对应，日志、指标和路由判断都会更顺。

再往下，就是 `to_thread()`。这几乎是你从 Java / 企业集成转到 Python Agent 时必补的一课。因为现实并不会因为你选择了 `asyncio`，就自动把周围世界都变成异步：

1. 你可能要接一个只有同步接口的客户 SDK。
2. 你可能要读写本地文件或压缩包。
3. 你可能要调某些老驱动、脚本包装器、阻塞式 HTTP 客户端或 CLI 适配层。

如果你在协程里直接调用这些阻塞函数，事件循环会被卡住。表面看只有一个任务慢了，实际上是整个宿主的并发能力被一起拖慢。`to_thread()` 的意义，不是把同步代码 magically 变成异步，而是把阻塞从事件循环线程挪走，让宿主还能继续调度其他协程。

这一点和你的历史经验也很能对上。你以前不会把一个可能阻塞 10 秒的客户驱动塞进 Tomcat 主线程里硬跑；你会隔离线程池、做超时、做舱壁。`to_thread()` 在 `asyncio` 宿主里，就是一种轻量级的“阻塞隔离舱”。而且文档提到 `contextvars` 会传播过去，这对 Agent 系统尤其关键，因为 trace id、run id、tenant id、用户上下文这类变量可以继续贯穿日志链路，不至于一进线程就丢。

把这三者合起来，你会得到一个非常实用的 Agent 工程套路：

1. 先用 `Semaphore` 或 `BoundedSemaphore` 给工具调用加并发上限。
2. 再用 `as_completed()` 让已完成的结果尽早进入后续处理，而不是被慢任务拖住。
3. 如果某个步骤必须调用阻塞式库，就用 `to_thread()` 隔离它，避免卡死整个事件循环。

这套组合，本质上就是把“批量并发”升级成“受控并发”。对 Agent demo 来说，前者已经够了；对能上线、能压测、能接真实客户环境的宿主来说，后者才是门槛。

## 3 key takeaways

1. `Semaphore` 不是为了炫技，而是给 Agent 工具调用加一层明确的并发许可证，避免把下游接口和本机资源一起打爆。
2. `as_completed()` 适合“谁先完成谁先处理”的 Agent 子任务收割场景，比一把 `gather()` 等全部结束更贴近真实宿主需求。
3. `to_thread()` 是阻塞型遗留库接入 `asyncio` 的隔离带，尤其适合客户 SDK、文件 IO、旧驱动这类你短期内改不了、但又必须接的同步能力。

## relation to Agent engineering

这篇内容和 Agent 工程的关系非常直接。

第一层，是工具调用限流。一个 Agent 可能会并发发起检索、浏览器操作、数据库访问、外部 API 请求。如果没有 `Semaphore` 一类的许可证控制，很容易把下游速率限制、连接池上限、浏览器实例数打穿。

第二层，是结果收割策略。Agent 系统里很多步骤不是“等所有子任务都完再统一处理”，而是“哪个先回来就先推进”。`as_completed()` 很适合做多路检索、候选工具探测、并发校验、健康检查回收。

第三层，是阻塞库桥接。你接下来不可能永远只活在纯异步、全新栈里。真实客户环境里总会有同步 SDK、内部封装库、CLI 或文件流水线。`to_thread()` 让你能在不重写整个依赖世界的前提下，把它们接进 Agent 宿主。

第四层，是你的迁移优势。你过去做 ToB 集成时，本来就擅长在复杂边界里控制连接数、线程数、响应顺序和隔离策略。今天这组 Python 官方文档，本质上是在帮你把原有工程判断翻译成 `asyncio` 语法。

## a small action for tonight

今晚做一个 30 到 40 分钟的小实验：

1. 写 6 个模拟工具调用，其中 2 个很快、2 个中等、2 个故意慢。
2. 用 `asyncio.BoundedSemaphore(3)` 包住调用逻辑，确认同一时间最多只有 3 个任务真正进入执行区。
3. 用 `asyncio.as_completed()` 按完成顺序打印结果，观察快任务是否能先输出，而不是等最慢任务。
4. 再故意把其中一个“工具”改成同步阻塞函数，分别试一次直接调用和 `asyncio.to_thread()` 包装，感受事件循环是否被拖住。
5. 最后给每个任务打印一个 `request_id` 或 `trace_id`，验证切到 `to_thread()` 后上下文信息是否还能继续传递。

## 原文关键段落翻译（人工翻译，放在文末）

1. `asyncio` 的同步原语和 `threading` 的思路相似，但它们不是线程安全对象；同时，这些原语自身不提供超时参数，若要超时应在外层配合 `asyncio.wait_for()`。
2. `Semaphore` 维护一个内部计数器；每次获取许可证时计数减一，每次释放时计数加一；当计数为 0 时，新的获取操作会阻塞，直到其他任务释放许可证。
3. 普通 `Semaphore` 允许释放次数超过获取次数；`BoundedSemaphore` 则会在释放后计数超过初始值时抛出 `ValueError`。
4. `as_completed()` 会并发运行一组 awaitable，并按照它们完成的先后顺序返回结果；如果设定了超时且不是所有任务都在时限内完成，会抛出 `TimeoutError`。
5. `to_thread()` 会把阻塞函数放到单独线程中执行，从而避免阻塞事件循环；当前的 `contextvars` 上下文也会被传播到该线程。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.python.org/3/library/asyncio-sync.html
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.python.org/3/library/asyncio-task.html
