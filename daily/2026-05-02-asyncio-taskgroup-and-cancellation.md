# title

2026-05-02 为什么 Agent 工程要尽快补上 `asyncio`、`TaskGroup` 和取消语义

## original source

- 标题：`asyncio` — Asynchronous I/O | Python 3.14.4 documentation
- 链接：https://docs.python.org/3/library/asyncio.html
- 来源类型：官方文档
- 访问时间：2026-05-02

- 标题：Coroutines and tasks | Python 3.14.4 documentation
- 链接：https://docs.python.org/3/library/asyncio-task.html
- 来源类型：官方文档
- 访问时间：2026-05-02

## why read today

你前几天已经补了 `pyproject.toml`、`pytest` 布局、fixtures、monkeypatch，也看过 MCP、Eval、Tracing。下一步如果还停留在“会写同步 Python 脚本”的层面，到了真正的 Agent 工程阶段就会开始卡住，因为很多核心能力天然建立在异步 IO 上：

1. 一个 Agent 同时调多个工具、等多个外部响应，本质就是并发等待。
2. MCP client/server、HTTP 调用、流式输出、subprocess、队列消费，很多都和事件循环直接相关。
3. 你以后写的不是“一次性脚本”，而是需要稳定运行、能取消、能清理、能隔离失败的服务化代码。

对你这种做了 10 年 Java / Spring / IoT 集成 / ToB 交付的人来说，这不是从零学“并发”本身，而是把原来熟悉的线程池、Future、超时、资源清理、链路编排，迁移到 Python 的 `async` / `await` / event loop / task cancellation 这套心智模型里。

## original-text translation

Python 官方先在 `asyncio` 总览页里把定位讲得很清楚：`asyncio` 是 Python 标准库里用于编写并发代码的基础库，使用 `async/await` 语法，特别适合 IO 密集型和高层网络代码。它不只是“能异步请求一下接口”，而是给你一整套高层 API，用来并发运行协程、处理网络 IO、控制子进程、通过队列分发任务、同步并发代码；同时也提供更底层的 event loop、transport、protocol 等能力，供框架和库作者扩展。

在 `Coroutines and tasks` 这篇里，官方把几个最容易混淆的概念拆开了。第一，`async def` 定义的是 coroutine function，调用之后得到的是 coroutine object；仅仅“调用”协程并不会执行它，真正让它跑起来，要么用 `asyncio.run()` 作为顶层入口，要么在另一个协程里 `await` 它，要么把它包装成 `Task` 交给事件循环调度。

官方随后用例子说明：如果连续 `await` 两个协程，它们是串行等待的；如果用 `asyncio.create_task()` 把两个协程包成任务再等待，它们就能并发推进。更进一步，Python 3.11 引入 `asyncio.TaskGroup` 作为更现代的方式：你在 `async with TaskGroup()` 里创建一组相关任务，退出上下文时会隐式等待所有任务完成，而且它对失败传播和取消处理有更强的安全保证。

文档里还有两个工程上非常关键的提醒。第一，`create_task()` 创建出来的任务要保留引用，否则任务可能因为事件循环只保留弱引用而在执行中途被回收；这对“随手 fire-and-forget 一下”的代码是个很实际的坑。第二，任务取消不是边角料，而是 `asyncio` 结构化并发的一部分：任务被取消时会抛出 `CancelledError`，协程通常应该用 `try/finally` 做清理，并在清理后继续把取消传播出去；如果把取消吞掉，`TaskGroup`、`timeout()` 这类依赖取消机制的组件就可能表现异常。

## Chinese deep summary

如果把今天这两篇官方文档压缩成一句话，那就是：在 Agent 工程里，`asyncio` 不是“性能优化技巧”，而是“系统如何同时等待多个外部世界并维持边界”的基础运行模型。

这点为什么对你特别重要？因为你原来的经验里，系统集成的大头并不在 CPU 计算，而在“等”：等接口、等设备、等消息、等数据库、等下游系统确认。Agent 系统也是同一类问题，只不过等待对象从“企业系统接口”扩展成了“模型响应、工具调用、MCP server、搜索结果、文件系统、网络服务”。如果没有异步心智，你很容易把 Agent 项目写成一串同步阻塞调用：功能能跑，但并发差、可取消性差、资源清理差、调试时也很难看清楚谁在等谁。

对 Java 背景的人，一个常见误区是把 Python 异步简单类比成“轻量线程”。这个类比不完全错，但不够准确。`asyncio` 的关键不是“多线程替代品”，而是 cooperative scheduling，也就是任务在 `await` 时显式让出执行权，事件循环再去推进别的任务。它更像你把原来隐式藏在线程调度里的等待点，显式拉到了代码层面。这样做的好处是：IO 密集场景下开销更低、控制粒度更细、组合能力更强；代价是你必须对“哪里会阻塞、哪里要 await、哪里能并发、哪里要清理取消”保持清醒。

`TaskGroup` 这块尤其值得你重点记。很多入门资料都先教 `create_task()`，但真正进入工程阶段，`TaskGroup` 往往更接近你想要的语义。因为 Agent 的并发任务通常不是彼此毫无关系的后台小任务，而是一组“共同组成一次请求处理”的子步骤。比如：

1. 同时请求两个检索源；
2. 并发拉取两个 MCP 工具结果；
3. 一边等模型输出，一边准备 tracing / logging / usage 统计；
4. 为一个用户请求启动若干并行子任务，然后在超时或失败时整体收拢。

这种场景里，最危险的往往不是“慢一点”，而是“有的子任务失败了，其它子任务还在后台偷偷跑”，最后造成资源泄漏、日志混乱、重复请求、甚至脏状态。`TaskGroup` 解决的就是这个问题：它把“一组相关任务的生命周期”绑在一起，只要其中一个任务以非取消异常失败，其余任务会被取消，最后把异常成组抛出来。这种结构化并发思路，和你做企业交付时强调的“一个业务事务里的子步骤要么一起收敛、要么整体失败回滚”非常接近。

取消语义是第二个不要轻视的点。很多人第一次写异步代码时，会把取消当成“用户不想要了，随便停一下”。但在 Agent 工程里，取消其实是正常控制流的一部分。举例说：

1. 用户关闭页面，请求应立即停止；
2. 总超时到了，仍未完成的工具调用应被撤掉；
3. 某个并行分支已经失败，其它分支没有继续跑下去的意义；
4. 上游服务断开，当前请求链路需要快速清场。

如果你的协程没有正确处理 `CancelledError`，最常见的后果不是立刻报错，而是留下半拉子状态：还没关闭的连接、还没清理的临时文件、还在持续写日志的后台任务、还在继续消耗 token 或调用配额的外部请求。官方文档建议用 `try/finally` 来兜底清理，并且一般不要吞掉取消，这是非常“工程化”的建议，不是什么语法细节。

另外，文档里“要保存 `create_task()` 返回值”的提醒也特别值得放进你的长期习惯。做 Java 的人通常默认只要任务提交到执行器就“有人托管”，但 `asyncio` 不是这个模型。事件循环对任务只保留弱引用，意味着你如果只是临时 `create_task()` 却不保存引用，任务理论上可能在结束前被回收。这对写 Agent 背景任务、trace flush、异步日志落盘、非阻塞预取这类逻辑时很关键。正确做法不是到处裸奔式地 `create_task()`，而是：

1. 能放进 `TaskGroup` 的放进 `TaskGroup`；
2. 真要后台运行的任务，用集合或专门管理器显式持有；
3. 对完成后的任务做回收，避免集合无限增长。

把这些点拼起来，你会看到一个很清晰的迁移路径：`pyproject` 和测试体系解决的是“项目怎么站住”；`asyncio` 解决的是“运行时怎么正确并发”；后面你再去看 FastAPI、MCP SDK、Agent runtime、streaming、observability，理解会顺很多。否则很容易出现一种假熟练：API 能调，Demo 能跑，但一到真实多工具、多请求、多超时、多取消的场景，系统行为就开始不可控。

## 3 key takeaways

1. `asyncio` 是 Python 里处理 IO 密集并发和高层网络代码的基础运行模型；对 Agent、MCP、流式调用来说，它更像“地基”而不是“高级技巧”。
2. `TaskGroup` 比裸 `create_task()` 更适合表达“一组相关子任务共同构成一次请求处理”的工程语义，能更安全地收拢失败和取消。
3. 取消、清理、任务引用管理都不是细节；如果这些边界没处理好，Agent 系统最容易出现幽灵任务、资源泄漏和难复现的问题。

## relation to Agent engineering

这篇内容和 Agent 工程的关系非常直接。

第一层是工具并发。一个 Agent 很少只调用一次工具就结束，更常见的是多个工具并行探测、多个候选信息源同时拉取、或者一边调用工具一边等待模型继续决策。这里天然需要 `await`、任务调度和超时控制，而不是串行阻塞。

第二层是 MCP。无论你后面是接 stdio 还是基于网络 transport，MCP 的很多交互都是异步等待外部进程或远端响应。你如果不理解协程、任务组、取消，后面一旦出现“客户端超时了但 server 子进程还挂着”“上游请求结束了但工具还在跑”这类问题，会很难排。

第三层是 Eval 和 Observability。跑 eval batch、并发执行 case、采集 trace、设置统一 timeout，本质也依赖任务编排。你前几天学的 tracing，会在这一步真正落到代码运行模型里：不是只会看 trace，而是知道哪些 span 对应哪个 task，失败时哪些任务应该一起取消。

第四层是你自己的能力迁移。你不是缺工程意识，而是要把原来在线程池、Future、事务边界、超时传播、资源释放上的经验，翻译成 Python 异步世界的写法。今天这两篇官方文档就是最适合做这次翻译的入口。

## a small action for tonight

今晚花 25 分钟做一个极小练习，不追求完整 Agent，只练异步基本功：

1. 写一个 `async def` 函数，内部 `await asyncio.sleep()` 模拟两个外部工具调用。
2. 先用连续 `await` 跑一版，再改成 `TaskGroup` 并发跑一版，实际打印开始时间和结束时间，对比总耗时。
3. 再故意让其中一个任务抛异常，观察 `TaskGroup` 下另一个任务会不会被取消，并在 `finally` 里打印清理日志。

如果今晚只记住一句话，就记这句：Agent 工程里的异步，重点不是“更快”，而是“把并发等待、失败传播和资源清理做对”。

## 原文关键段落翻译（人工翻译，放在文末）

1. `asyncio` 是一个使用 `async/await` 语法来编写并发代码的库，它常被多个异步框架作为基础，尤其适合 IO 密集型和高层网络代码。
2. `asyncio` 提供的高层 API 包括：并发运行 Python 协程、执行网络 IO 和进程间通信、控制子进程、通过队列分发任务、以及对并发代码进行同步。
3. 仅仅调用一个协程并不会调度它执行；要真正运行协程，需要使用 `asyncio.run()`、在另一个协程里 `await` 它，或者把它包装成 `Task`。
4. `asyncio.create_task()` 会把协程包装成任务并安排执行；而 `TaskGroup` 是更现代的替代方案，适合等待一组相关任务完成，并提供更强的安全保证。
5. 任务被取消时，`CancelledError` 会在下一个机会点抛出；协程通常应使用 `try/finally` 做稳健清理，并在清理完成后继续传播取消。
6. `create_task()` 返回的任务应保存引用；因为事件循环只保留弱引用，没有其它引用的任务可能在完成前被垃圾回收。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.python.org/3/library/asyncio.html
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.python.org/3/library/asyncio-task.html
