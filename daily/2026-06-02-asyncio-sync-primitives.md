# title

2026-06-02 为什么 Agent 工程里不能把 `asyncio.Lock` 当成“Java synchronized 的低配版”

## original source

- 标题：`Synchronization Primitives`
- 链接：https://docs.python.org/3/library/asyncio-sync.html
- 来源类型：Python 官方文档
- 访问时间：2026-06-02

- 标题：`PEP 3156 – Asynchronous IO Support Rebooted: the “asyncio” Module`
- 链接：https://peps.python.org/pep-3156/
- 来源类型：Python 标准提案 / asyncio 设计规范
- 访问时间：2026-06-02

## why read today

这几天你的阅读主线已经补到了 `TaskGroup`、超时、取消、队列背压、子进程、JSON Schema、MCP 协议和 eval。下一步最该补的，不是再堆一个新框架，而是把 **单线程异步系统里的“状态协调”** 这件事补扎实。

这块对你尤其重要，因为你有 10 年 Java / Spring / IoT 集成 / ToB 交付背景。你对锁、条件变量、线程池、消费者并发这些概念并不陌生，但在 Python `asyncio` 里，问题的形态变了：

1. 不是多个 OS 线程抢共享对象，而是多个 coroutine 在同一个 event loop 里交错推进。
2. 不是“线程安全容器”优先，而是“谁拥有状态、谁负责 await 边界、谁决定取消和唤醒”优先。
3. Agent 工程里真正会出事故的地方，往往不是模型调用本身，而是工具并发、会话状态、缓存填充、浏览器任务、MCP 请求队列这些外围协调层。

如果这层语义不清楚，系统就很容易出现这几种典型问题：

1. 两个 tool run 同时写同一个 session state，结果 trace 看起来正常，但最终状态被后写覆盖。
2. 一个后台缓存填充任务本来只该跑一次，结果并发命中时被重复触发三四次。
3. 你以为自己限制了并发，实际上只是控制了调用点，没有控制共享资源访问。
4. 你把 Java 里的“锁住一段代码”直觉照搬过来，但在 `await` 穿插下，锁的粒度和等待点完全不是原来的味道。

所以今天这篇，补的是 Agent runtime 最底层但最实用的一层：`Lock`、`Event`、`Condition`、`Semaphore` 到底分别解决什么问题，什么时候该用，什么时候不该用。

## original-text translation

Python 官方这页先把一个很容易被忽略的前提说死了：`asyncio` 里的同步原语虽然设计得和 `threading` 模块相似，但有两个关键限制。第一，它们 **不是线程安全的**，不能拿来做 OS 线程之间的同步；第二，这些原语的方法 **不接受 timeout 参数**，如果你要给等待操作加超时，要在外层配合 `asyncio.wait_for()`。

文档接着定义了几种基础原语。`Lock` 是最基本的互斥锁，用来保证对共享资源的排他访问；而且从 Python 3.10 起，获取锁是公平的，多个 coroutine 等待时，最先等待的会最先得到锁。官方推荐用 `async with lock:` 来拿锁和释放锁，这样即使中间抛异常，也不会把锁遗留在已持有状态。

`Event` 更像一个“一次广播开关”。内部有一个布尔标志，初始为 false；调用 `set()` 后，所有等待这个事件的任务都会被立刻唤醒；再次 `clear()` 后，又回到未触发状态。它适合表达“某个条件已经发生”，而不是“某个资源只能一个人用”。

`Condition` 适合更复杂的等待场景。它把一个 `Event` 的通知能力和一个 `Lock` 的互斥保护绑在一起，让你能在持有锁的前提下等待某个谓词成立。等待时会临时释放底层锁，等到被唤醒后再重新获取，然后继续往下执行。它不是简单的“通知一下”，而是“在受保护状态下等待某个条件变成真”。

`Semaphore` 则不是排他，而是限流。内部维护一个计数器，每次 `acquire()` 计数减一，`release()` 计数加一；计数为零时，新的等待者只能阻塞。它最适合做“最多同时允许 N 个协程进入某段逻辑”，例如限制并发工具调用数、浏览器 worker 数、数据库连接使用数。

PEP 3156 则给了这些原语的设计背景。这个提案说明：`asyncio` 的锁、事件、条件变量、信号量和队列，在概念上尽量对应线程版同类对象，但阻塞方法改成了 coroutine，并且默认不提供 timeout 参数，需要用 `asyncio.wait_for()` 在外层追加等待边界。换句话说，`asyncio` 不是否定你熟悉的并发概念，而是把它们搬进 event loop 语义里重做了一遍。

## Chinese deep summary

今天这两个官方来源，核心不是在教你 API 名字，而是在帮你完成一个认知切换：

**在 Python Agent 工程里，“同步”主要不是解决多线程抢 CPU，而是解决多个协程在共享状态、共享配额、共享生命周期时如何有序协作。**

第一层，要先把“为什么单线程还需要锁”这件事想透。

很多 Java 背景的人第一次看 `asyncio.Lock`，会本能觉得奇怪：都单线程 event loop 了，为什么还要锁？问题在于，单线程不等于单步骤。协程会在 `await` 点让出执行权，于是两个任务虽然不并行执行 CPU 指令，但会交错修改同一份状态。

对 Agent 系统来说，这太常见了：

1. 两个用户请求同时命中同一个缓存填充逻辑。
2. 一个会话里同时触发两个 tool call，都会写 `session.memory`。
3. 一个 trace span 关闭前，另一个协程已经开始复用同一份上下文对象。

这类问题不是“线程安全”意义上的 data race，但仍然是 **协程级别的状态竞争**。`Lock` 的作用，就是让“读改写”这一段在协程层面保持原子性。它保护的是 event loop 中的进入顺序，不是 CPU 级原子指令。

第二层，`Lock`、`Event`、`Condition`、`Semaphore` 对应的是四种完全不同的问题，不要混用。

`Lock` 解决的是“同一时刻只能有一个协程碰这份状态”。典型场景是：

1. 更新内存中的会话对象。
2. 首次懒加载某个昂贵资源。
3. 给 trace / metrics 聚合器写共享缓冲区。

`Event` 解决的是“大家先别动，等某件事发生”。比如：

1. Agent runtime 等待模型客户端初始化完成。
2. Browser worker 等待登录状态准备好。
3. 某个 MCP 连接尚未 ready，多个请求先挂起等待。

这时你要的是广播唤醒，不是排他访问，所以用 `Lock` 就很别扭。

`Condition` 解决的是“在互斥保护下，等待某个状态变成真”。这比单纯 `Event` 更细。比如：

1. 等待某个共享队列从空变成非空。
2. 等待会话状态进入 `READY` 而不是 `BOOTING`。
3. 等待一个后台任务把结果写回后再继续。

你如果只靠 `Event`，可能只能知道“有人通知过”，却拿不到“通知时这份状态在锁保护下是可验证的”这层保证。

`Semaphore` 则是 Agent 工程里特别常用、但最容易被误当成“锁替代品”的工具。它不是保护共享状态，而是给共享容量设上限。比如：

1. 最多同时执行 4 个外部工具调用。
2. 最多同时打开 2 个浏览器页签抓取网页。
3. 最多同时跑 8 个 embedding / rerank 请求。

如果你用 `Lock` 去干这些事，只会把吞吐打成串行；如果你用 `Semaphore` 去保护一个复杂状态对象，又可能只是“限制人数”，并没有真正保证状态一致性。

第三层，官方特地强调“这些原语没有 timeout 参数”，这件事在工程上非常重要。

这不是标准库偷懒，而是一种设计态度：**等待多久，是调用方的控制语义，不是锁对象本身的业务语义。**

这和你熟悉的很多 Java API 不一样。你以前可能会找：

1. `tryLock(timeout)`
2. `poll(timeout)`
3. `offer(timeout)`

而在 `asyncio` 里，官方更倾向让你写成外层的：

1. `await asyncio.wait_for(lock.acquire(), timeout=...)`
2. `await asyncio.wait_for(event.wait(), timeout=...)`
3. `await asyncio.wait_for(semaphore.acquire(), timeout=...)`

这背后的好处是，超时策略不被埋进底层对象，而是显式地写在 workflow 边界上。对 Agent 系统很关键，因为不同链路对等待上限的容忍度完全不同：

1. 用户前台请求可能只允许等 2 秒。
2. 后台补偿任务可以等 20 秒。
3. 预热流程甚至可以一直等到系统 shutdown。

第四层，从 Java/IoT 集成思维迁移过来时，最值得保留的不是具体 API 经验，而是“资源所有权”和“并发边界”意识。

你原来做系统集成时，已经很熟悉这些问题：

1. 一个连接池允许多少并发。
2. 一个设备通道同一时刻是否只能有一个命令飞行中。
3. 一个订单状态更新是否必须串行。
4. 一个补偿任务完成前，主流程该不该继续。

这些问题在 Agent 系统里一个都没消失，只是对象变成了协程、工具、上下文、MCP session、browser page 和模型请求。今天这篇最有价值的地方，是帮你把老经验迁移到对的抽象层，而不是继续把注意力只放在模型和提示词上。

第五层，落到 Agent engineering，最常见的正确姿势其实很朴素。

1. 有共享可变状态，用 `Lock`。
2. 有“等某件事发生”的广播时机，用 `Event`。
3. 有“等状态满足某个条件且必须在互斥保护下检查”的需求，用 `Condition`。
4. 有“最多同时允许 N 个任务”的容量限制，用 `Semaphore`。
5. 有等待上限，再在外层加 `asyncio.wait_for()` 或结构化的 `asyncio.timeout()`。

只要这几条分得清，很多“偶发脏状态”“偶发重复执行”“偶发工具洪峰”问题，都会在设计阶段就少掉一大截。

## 3 key takeaways

1. `asyncio` 的同步原语解决的是协程之间的状态协调，不是线程之间的同步；它们不线程安全，但对单线程 event loop 中的共享状态仍然非常关键。
2. `Lock / Event / Condition / Semaphore` 分别对应排他访问、广播通知、受保护条件等待、并发容量限制，问题模型不同，不能互相替代。
3. 这些原语默认没有 timeout 参数，说明超时应该由外层 workflow 控制；在 Agent 工程里，这正好有利于把 SLA、取消和降级策略写在清晰边界上。

## relation to Agent engineering

这篇内容和 Agent engineering 的关系非常直接，而且是“写代码时立刻能用”的那种直接。

第一，它会影响你怎么设计 tool executor。很多工具不是不能并发，而是“最多并发 N 个”或者“同一 session 内要串行”。前者是 `Semaphore`，后者通常是 `Lock`，不要混。

第二，它会影响你怎么设计 session state。Agent 一旦开始有 memory、缓存、trace context、browser state、MCP connection state，就一定会碰到共享可变对象。你如果没有协程级互斥，问题不会天天爆，但一爆就很难查。

第三，它会影响你怎么桥接 Java 经验。你过去在 Spring Integration、消息消费、设备指令排队、连接池容量控制里学到的并发边界意识是有价值的；只是现在不要再默认映射到线程池，而要映射到 coroutine 生命周期和 event loop 调度。

第四，它和 MCP 也有直接关系。一个 MCP client 或 server 只要开始维护连接状态、资源缓存、请求队列、并发调用上限，就会进入这些原语的使用场景。协议层规范讲的是消息格式，真正把系统跑稳，靠的是这些运行时协调细节。

## a small action for tonight

今晚做一个 30 分钟的小实验，不用接模型，也不用接 MCP，纯练运行时语义：

1. 写一个内存里的 `session_state = {"counter": 0}`，启动 20 个协程，每个协程都先 `await asyncio.sleep(0)` 再做读改写；先不加锁，观察最终计数不稳定。
2. 再加 `asyncio.Lock` 把读改写包起来，确认结果稳定为 20。
3. 写一个 `asyncio.Semaphore(3)`，让 10 个伪工具任务并发执行，打印进入和退出时间，直观看到“最多 3 个同时运行”。
4. 再写一个 `Event`，让多个 worker 先 `await event.wait()`，主协程 2 秒后 `set()`；感受“广播式放行”和互斥锁完全不是一回事。
5. 最后给其中一个等待操作加上 `asyncio.wait_for()`，把“同步原语负责协调，超时由外层控制”这条原则固定下来。

如果你把这 4 个实验都跑通，后面再去写 Python Agent runtime、tool bridge、browser worker、MCP adapter，很多并发问题会提前变得可解释。

## 原文关键段落翻译（人工翻译，放在文末）

1. `asyncio` 的同步原语被设计得与 `threading` 模块中的同名对象相似，但有两个重要限制：它们不是线程安全的；而且这些方法不接受 `timeout` 参数，若需要超时应配合 `asyncio.wait_for()`。
2. `Lock` 实现的是互斥锁，用来保证对共享资源的排他访问；推荐用 `async with` 方式使用，这样无论代码块怎样退出，都能正确释放锁。
3. 获取锁是公平的：当多个协程都在等待同一把锁时，最终会由最早开始等待的那个协程先拿到锁。
4. `Event` 管理一个内部标志位；调用 `set()` 会唤醒所有等待该事件的任务。
5. `Condition` 结合了 `Event` 和 `Lock` 的能力，适合在受保护的共享状态上等待某个条件成立。
6. `Semaphore` 管理一个内部计数器；`acquire()` 会减一，计数为零时新的获取请求会阻塞，直到别的任务 `release()`。
7. PEP 3156 说明，这些锁、事件、条件变量、信号量和队列在概念上对应线程版对象，但阻塞方法改成了 coroutine，并且不直接提供 timeout，超时要由外层用 `asyncio.wait_for()` 附加。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.python.org/3/library/asyncio-sync.html
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://peps.python.org/pep-3156/
