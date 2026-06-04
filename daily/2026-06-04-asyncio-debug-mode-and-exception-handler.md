# title

2026-06-04 为什么你做 Agent 宿主时，必须尽早打开 `asyncio` 调试模式并接管事件循环异常

## original source

- 标题：`Developing with asyncio — Python 3.14.5 documentation`
- 链接：https://docs.python.org/3/library/asyncio-dev.html
- 来源类型：Python 官方文档
- 访问时间：2026-06-04

- 标题：`Event loop — Python 3.14.5 documentation`
- 链接：https://docs.python.org/3/library/asyncio-eventloop.html
- 来源类型：Python 官方文档
- 访问时间：2026-06-04

## why read today

你这几天已经连续补了 `asyncio` 的取消、超时、同步原语，以及 `Task` 和执行器 `Future` 的边界。下一步最该补的，不是再记一个 API，而是补上一个更接近生产现场的问题：

1. 协程宿主卡顿时，第一时间靠什么发现。
2. 某个任务异常但没人取结果时，怎么别让它悄悄烂在后台。
3. 工具调用、MCP bridge、trace flush、日志输出把 event loop 拖慢时，怎么尽早暴露。

这件事对你尤其重要，因为你有 10 年 Java / Spring / IoT 集成 / ToB 交付背景。你对“线程池监控、慢调用、未处理异常、统一异常兜底”这些概念非常熟，但到了 Python `asyncio`，很多人会不自觉退回 demo 心态：代码能跑就先跑，异常先打印，卡顿先凭感觉查。

这在 Agent 工程里很危险。因为 Agent 宿主通常同时承担：

1. 模型请求编排
2. tool 调用
3. MCP client / server 通信
4. trace / metrics / logging
5. 会话状态和审批恢复

一旦 event loop 层的慢回调、错误线程调用、未被消费的任务异常没有尽早暴露，系统就会出现一种非常典型的“表面能用、实则半失控”的状态。今天这两篇官方文档，补的正是这种运行时可诊断性。

## original-text translation

Python 官方在 `Developing with asyncio` 里先说明：`asyncio` 默认是生产模式，但为了开发和排错，提供了 debug mode。打开方式有四种：

1. 设环境变量 `PYTHONASYNCIODEBUG=1`
2. 开启 Python Development Mode
3. `asyncio.run(..., debug=True)`
4. 调 `loop.set_debug()`

官方接着强调，打开 debug mode 以后，不只是“多打一点日志”，而是会启用几类对工程非常关键的检查：

1. 许多非线程安全的 `asyncio` API，如果从错误线程调用，会直接抛异常。
2. I/O selector 执行过慢时会被记录。
3. 执行超过 100 毫秒的 callback 会被记录；阈值可以通过 `loop.slow_callback_duration` 调整。

同一页文档还提醒几个非常常见的坑。第一，阻塞型代码不能直接跑在 event loop 线程里，否则整个协程系统都会被一起拖慢；这类代码应该放到 executor 里。第二，网络日志本身也可能阻塞 event loop，官方建议把日志处理放到单独线程，或者使用非阻塞 I/O。第三，如果一个协程函数被调用了，却既没有 `await`，也没有被 `create_task()` 调度，`asyncio` 会发出 `RuntimeWarning`。第四，如果一个任务抛出了异常，但这个异常从来没有被取回，`asyncio` 会在对象被垃圾回收时记录 “Task exception was never retrieved”；而 debug mode 能进一步给出“这个任务是在哪里创建出来的”。

`Event loop` 文档则把事件循环异常处理接口讲得更底层。官方提供 `loop.set_exception_handler(handler)` 来替换默认异常处理器；自定义处理器签名是 `(loop, context)`。这里的 `context` 不是随便一段字符串，而是一个包含运行时细节的字典，可能带有：

1. `message`
2. `exception`
3. `future`
4. `task`
5. `handle`
6. `protocol`
7. `transport`
8. `socket`
9. `source_traceback`
10. `handle_traceback`
11. `asyncgen`

文档还说明了两个很值得记住的点。第一，如果异常是代表某个 `Task` 或 `Handle` 触发的，异常处理器会运行在那个任务或回调句柄对应的 `contextvars.Context` 里。第二，`loop.default_exception_handler(context)` 可以在自定义处理器内部继续调用，也就是说你可以先补充自己的 trace / run_id / tool_name 记录，再把默认日志行为保留下来。

把这两篇合起来看，官方真正想传达的是：`asyncio` 的稳定性不是靠“出事后翻控制台”，而是靠运行时在第一时间暴露错误线程访问、慢回调、未 await 协程、未取回异常，以及通过统一异常入口把上下文打全。

## Chinese deep summary

今天这组材料，最值得你记住的一句话是：

**对 Agent 宿主来说，`asyncio` debug mode 和 event loop exception handler，不是“开发时顺手开一下”的辅助功能，而是运行时可诊断性的最低配置。**

第一层，为什么它对 Agent 宿主比对普通脚本更重要？

因为 Agent 系统不是只跑一个 `await llm_call()`。一旦进入真实工程，你很快就会同时有：

1. 多个协程并发调工具；
2. 某些工具经由线程池桥接阻塞 SDK；
3. MCP 连接和消息收发；
4. trace、日志、审计、指标异步落盘；
5. 用户中断、审批恢复、超时取消。

这意味着一条请求路径里会有很多“不是主业务结果，但一旦出问题就拖垮宿主”的外围动作。Java 里你会给线程池、慢 SQL、未捕获异常、日志阻塞配监控；到了 Python `asyncio`，这些问题没有消失，只是载体变成了 event loop、Task、callback 和 executor。

第二层，debug mode 的真正价值，不是“更详细”，而是“更早暴露错误边界”。

很多异步 bug 最难受的地方就在于，它不是立刻把程序打崩，而是先让系统进入一种模糊异常状态。例如：

1. 某个同步日志 handler 偶尔卡 300ms，导致 event loop 周期性抖动；
2. 某个协程创建后忘了 `await` 或忘了 `create_task()`，功能偶发缺失但主流程没报错；
3. 某个后台任务抛异常，但没人等待它结果，用户表面看起来只觉得“偶尔不稳定”；
4. 某个别的线程直接动了 loop 对象，在线上变成偶发 race condition。

debug mode 之所以值钱，是因为它把这些“容易拖成玄学排障”的问题，尽量提前变成明确的信号：

1. 错线程调用直接报错；
2. 慢 callback 被点名；
3. selector 太慢会留痕；
4. 未 await / 未取回异常会给出更接近源头的信息。

这和你熟悉的 ToB 交付经验其实完全一致。真正让人崩溃的不是“系统有 bug”，而是“系统出问题时不给你足够早、足够准的信号”。`asyncio` debug mode 干的，就是把这些信号往前提。

第三层，为什么“网络日志可能阻塞 event loop”这句话值得单独记？

因为很多人做 Agent 宿主时，会很自然地想把 trace、token usage、tool 输入输出、MCP 往返消息都即时打到远端日志、APM 或观测服务里。但官方明确提醒：**日志本身也可能成为阻塞源**。

这件事很像你过去做 Java 系统时遇到的：

1. 同步 appender 拖慢请求线程；
2. 下游日志平台抖动，把业务线程一起拖住；
3. 看起来是“业务接口慢”，实际是观测链路反噬主链路。

在 Python `asyncio` 里，后果往往更集中，因为 event loop 是协程宿主。一旦把它卡住，受影响的不是一个线程里的一个请求，而是整批协程调度。因此，Agent runtime 里的日志、trace 导出、审计上报，最好天然按“非阻塞优先，必要时单独线程/队列解耦”的思路设计。

第四层，`set_exception_handler()` 的价值，在于给了你一个统一的“宿主级异常入口”。

很多 Agent 系统后面都会演变出这样的需求：

1. 异常日志里必须自动带 `run_id`；
2. 如果当前是在某个 tool span 里，还要带 `tool_name`；
3. 如果错误来自某个 transport / protocol，要能区分是 MCP 通信层还是工具业务层；
4. 对某些异常要补指标、补 trace event、补审计记录；
5. 默认异常日志不能丢，因为它还包含底层 runtime 提供的细节。

`loop.set_exception_handler()` 正好就是干这个的。它不是替代业务层 `try/except`，而是在 event loop 层给你一个统一兜底点。你可以把它理解成“`asyncio` 世界里的宿主级异常钩子”，很像你在 Java 里做统一异常映射、线程池异常采集、或框架级错误监听。

第五层，`context` 字典为什么重要？

因为它告诉你，事件循环级异常并不只是“一段报错字符串”。`context` 里可能直接给到 `task`、`future`、`handle`、`protocol`、`transport`、`socket`、`source_traceback`。这意味着：

1. 你能知道错的是任务、回调、协议还是传输；
2. 你能把异常挂回当前 trace；
3. 你能更容易定位“这个后台任务到底从哪儿创建出来的”；
4. 你能把异常分类成“业务失败”和“宿主失稳”两大类。

这对你很关键，因为你不是只想学会“把 Agent 跑起来”，而是想做成可交付、可排障、可复盘的系统。真实客户环境里，最值钱的不是一句“报错了”，而是你能不能很快回答：

1. 是哪个 run 出的问题；
2. 问题发生在模型、工具、MCP、日志、还是宿主线程桥接层；
3. 这是业务异常，还是 runtime 失稳信号；
4. 有没有扩大影响到其他并发任务。

第六层，这套思路怎么和你现有 Java 经验桥接？

可以直接这样迁移：

1. `asyncio` debug mode 很像把线程边界检查、慢调用提示、开发期告警前置打开；
2. `set_exception_handler()` 很像框架级统一异常入口；
3. `loop.slow_callback_duration` 很像给 event loop 的“慢执行阈值”；
4. “未 retrieved 异常” 很像后台任务失败但没有被主流程消费，最后只能在宿主兜底日志里暴露。

区别只是：Java 常围绕请求线程、线程池和阻塞调用看问题；Python Agent 宿主则更常围绕 event loop、Task、callback 和 executor 桥接看问题。

第七层，如果只记一组最落地的规则，可以记这四条：

1. 开发和联调阶段，默认打开 `asyncio` debug mode。
2. 任何远端日志、trace、审计上报，都先假设它可能阻塞 event loop。
3. 给宿主注册统一的 `loop.set_exception_handler()`，把 `run_id`、`tool_name`、trace 信息补齐后再回落到默认处理器。
4. 看到 “coroutine was never awaited” 或 “Task exception was never retrieved”，不要把它当日志噪音；这类信号在 Agent runtime 里通常对应真实的状态丢失或后台失败。

只要这四条先立住，你后面再接 browser worker、MCP adapter、tool orchestrator、trace exporter 时，很多“偶尔慢、偶尔丢、偶尔挂”的问题会更早变得可解释。

## 3 key takeaways

1. `asyncio` debug mode 的价值不只是多日志，而是尽早暴露错线程调用、慢 callback、未 await 协程和未取回任务异常这些高风险运行时信号。
2. `loop.set_exception_handler()` 提供的是宿主级统一异常入口；对 Agent 工程来说，它适合补 `run_id`、`tool_name`、trace 和异常分类，而不是只打印一条字符串。
3. 日志、trace、审计本身也可能阻塞 event loop；在 Python Agent 宿主里，观测链路必须按“不能反噬主调度线程”的原则设计。

## relation to Agent engineering

这篇内容和 Agent engineering 的关系非常直接。

第一，它决定你怎么做宿主稳定性。Agent runtime 的核心不是模型调用本身，而是围绕模型的并发调度、工具桥接、状态传播、异常收口和观测输出。debug mode 和统一异常处理，就是这层稳定性的最小起点。

第二，它决定你怎么做 tool 和 MCP 运行时排障。以后你接阻塞 SDK、远端 MCP server、浏览器驱动、文件系统工具时，很多失败不会以“主流程抛异常”这种最干净的方式出现，而会以后台任务异常、慢 callback、被卡住的日志 handler、错误线程访问等形式出现。你越早把这些入口接住，系统越接近可交付。

第三，它决定你怎么做 trace 关联。官方文档明确说，异常处理器可能运行在对应任务或回调的 `contextvars.Context` 里，这意味着你可以把已经建立好的 `run_id`、trace_id、tool 上下文接回异常日志，而不是靠字符串猜测。

第四，它也在补你从 Java 到 Python 的最后一段运行时迁移。你原来已经很熟悉“统一异常入口 + 慢调用暴露 + 线程边界约束 + 日志异步化”这一整套工程习惯。今天这篇的价值，就是把这套习惯落到 `asyncio` 宿主上，而不是让 Python Agent 工程停在脚本化阶段。

## a small action for tonight

今晚做一个 30 分钟的小实验，目标是把“能跑”提升成“能诊断”：

1. 写一个最小 `asyncio.run(main(), debug=True)` 脚本，并把 `logging.getLogger("asyncio").setLevel(logging.DEBUG)` 打开。
2. 故意写一个 `coro()` 只调用不 `await`，观察 `RuntimeWarning` 和 debug mode 下的创建栈信息。
3. 再写一个 `asyncio.create_task(bug())`，让 `bug()` 抛异常但不去 `await task`，观察 “Task exception was never retrieved”。
4. 注册一个 `loop.set_exception_handler()`，把 `context["message"]`、`context.get("exception")` 和你自定义的 `run_id` 打出来，再继续调用 `loop.default_exception_handler(context)`。
5. 最后模拟一个阻塞日志或慢 callback，例如在 callback 里 `time.sleep(0.2)`，确认慢回调日志会被打出来。

如果这 5 步跑顺，你就已经有了一个像样的 Python Agent 宿主诊断骨架。

## 原文关键段落翻译（人工翻译，放在文末）

1. `asyncio` 默认运行在生产模式；为了方便开发，它提供了 debug mode，可以通过环境变量、Python Development Mode、`asyncio.run(debug=True)` 或 `loop.set_debug()` 启用。
2. 开启 debug mode 后，许多非线程安全的 `asyncio` API 在被错误线程调用时会抛异常；I/O selector 过慢会被记录；执行超过 100 毫秒的 callback 也会被记录，而阈值可以通过 `loop.slow_callback_duration` 修改。
3. 阻塞型代码不应直接在 event loop 线程中运行；否则整个协程系统都会被拖慢，应改用 executor。
4. 网络日志可能阻塞事件循环；官方建议用单独线程处理日志，或使用非阻塞 I/O。
5. 如果协程函数被调用了，但既没有 `await`，也没有被 `asyncio.create_task()` 调度，`asyncio` 会发出 `RuntimeWarning`。
6. 如果某个 `Future` 或任务里的异常从未被用户代码取回，`asyncio` 会在对象被垃圾回收时记录相应日志；在 debug mode 下，还能看到任务创建位置。
7. `loop.set_exception_handler(handler)` 允许你自定义事件循环异常处理器；处理器签名是 `(loop, context)`。
8. 传给异常处理器的 `context` 是一个字典，可能包含 `message`、`exception`、`future`、`task`、`handle`、`protocol`、`transport`、`socket`、`source_traceback`、`handle_traceback` 和 `asyncgen` 等信息。
9. 如果异常是代表某个 `Task` 或 `Handle` 触发的，自定义异常处理器会运行在那个任务或回调句柄的 `contextvars.Context` 中。
10. 自定义异常处理器内部仍然可以调用 `loop.default_exception_handler(context)`，保留默认异常输出行为。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.python.org/3/library/asyncio-dev.html
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.python.org/3/library/asyncio-eventloop.html
