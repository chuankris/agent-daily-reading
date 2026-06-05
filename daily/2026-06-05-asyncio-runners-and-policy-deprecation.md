# title

2026-06-05 为什么现在就该把 Agent 宿主的事件循环入口迁到 `asyncio.run()` / `Runner`

## original source

- 标题：`Runners — Python 3.14.5 documentation`
- 链接：https://docs.python.org/3/library/asyncio-runner.html
- 来源类型：Python 官方文档
- 访问时间：2026-06-05

- 标题：`Policies — Python 3.14.5 documentation`
- 链接：https://docs.python.org/3/library/asyncio-policy.html
- 来源类型：Python 官方文档
- 访问时间：2026-06-05

## why read today

你这轮补课已经把 `asyncio` 的取消、超时、同步原语、`Task` 和调试模式补了不少。下一步最该补的，不再是“某个协程 API 怎么用”，而是 **Agent 宿主到底该怎么启动、持有和关闭事件循环**。

这件事对你尤其重要，因为你不是在写一次性脚本，而是准备做：

1. 带 CLI 入口的 Agent 工具。
2. 长时间运行的 MCP host / client。
3. 需要优雅退出、清理资源、保留 trace 的服务进程。

Python 3.14 已经把 `asyncio` policy system 标成 deprecated，并明确写出会在 Python 3.16 移除。也就是说，过去很多“先 `get_event_loop()` 再手动塞配置”的写法，已经不再是长期可维护路径。官方给出的迁移方向非常明确：

1. 程序主入口优先用 `asyncio.run()`
2. 需要复用同一个 loop 和 `contextvars.Context` 时用 `asyncio.Runner`
3. 需要定制 loop 时，通过 `loop_factory` 做，而不是继续依赖 policy

这篇内容很适合你当前阶段，因为它补的是“宿主骨架”。你原来做 Java / Spring / IoT 集成时，很清楚启动方式、线程模型、资源关闭顺序、Ctrl+C 中断语义，都会影响系统是否能交付。今天这两篇官方文档，补的正是 Python Agent 宿主的这层地基。

## original-text translation

`Runners` 文档先把官方推荐路径说得很直接：`asyncio.run()` 是运行 asyncio 程序的主入口。它负责创建事件循环、运行可等待对象、结束异步生成器、关闭默认 executor，并在最后关闭 loop。文档还特别补充，现在如果你想配置事件循环，推荐通过 `loop_factory` 来做，而不是再通过 policy system。

文档同时给出更适合“多次顶层调用”的方案：`asyncio.Runner`。它是一个 context manager，目的是让多个顶层 async 函数在同一个 event loop 和同一个 `contextvars.Context` 里运行。`Runner.run()` 可以接收一个 awaitable；如果传进来的是 coroutine，它会被包装成 `Task`。`Runner.close()` 负责收尾：结束异步生成器、关闭默认 executor、关闭 event loop、释放嵌入的 context。

`Runners` 文档还有一个对工程很重要、但很多人平时不会注意的点：`Runner.run()` 在执行用户代码前，会先安装自定义的 `SIGINT` 处理器。按官方描述，第一次 `Ctrl+C` 会先取消 main task，在 main task 内部触发 `CancelledError`，让 `try/finally` 之类的清理逻辑有机会跑完；之后 `Runner.run()` 再抛出 `KeyboardInterrupt`。这意味着，官方不是把中断当“粗暴打断”，而是尽量转成“先取消主任务，再走清理路径”。

`Policies` 文档则把旧路径的生命周期说得很明确：policy system 已经从 Python 3.14 开始弃用，并将在 Python 3.16 移除。包括 `asyncio.get_event_loop_policy()`、`asyncio.set_event_loop_policy()`、`AbstractEventLoopPolicy`、`DefaultEventLoopPolicy`，以及 Windows 下的 selector / proactor policy，都已经进入弃用通道。

同一页文档还强调了一个变化：默认 policy 的 `get_event_loop()` 在 Python 3.14 中，如果当前没有设置 event loop，会直接抛 `RuntimeError`。这意味着，很多过去靠“隐式取到一个 loop”勉强工作、但没有明确宿主管理边界的代码，会在新版本里更早暴露问题。

把两篇合起来看，官方真正传递的信息其实很集中：**以后 asyncio 程序的宿主管理，应从“全局 policy + 隐式 loop”迁到“显式入口 + 显式 loop_factory + 明确关闭语义”。**

## Chinese deep summary

今天这篇最值得你记住的一句话是：

**对 Agent 工程来说，`asyncio.run()` / `Runner` 不是“更现代的写法”，而是 Python 官方正在收紧的宿主边界。**

第一层，为什么这件事比再学一个异步 API 更重要？

因为 Agent 系统真正难的，不只是 `await` 一个模型请求，而是宿主怎么管住整条运行时链路：

1. 事件循环何时创建。
2. 顶层任务在哪个上下文里跑。
3. Ctrl+C 或容器停止时，先取消谁、后关闭谁。
4. 线程池桥接、日志导出、trace flush 有没有机会优雅收尾。

如果这些边界没立住，系统很容易变成“功能能跑，但退出脏、清理乱、偶发挂死”。这和你过去做 ToB 集成时看到的现象很像：问题不一定出在主业务，而出在启动和收尾。

第二层，为什么官方要弱化 policy、强化 `run()` / `Runner`？

因为 policy 本质上是“进程级全局配置对象”。它早期很方便，但工程上有几个天然问题：

1. 全局状态难推断，库代码和应用代码容易互相影响。
2. 不同线程、不同宿主边界下，谁负责当前 loop，不够直观。
3. 出现问题时，开发者常常分不清“这个 loop 是谁建的、谁该关、谁改过 policy”。

而 `asyncio.run()` / `Runner` 的思路正相反：把 loop 的创建、使用、关闭，收进一个明确作用域里。你可以把它理解成从“进程级魔法”回到“宿主显式管理”。

这对 Agent 工程很关键。因为 Agent runtime 通常要接很多外围设施：

1. MCP 连接。
2. browser / shell / file 等工具桥接。
3. trace 和 metrics 的 context 传播。
4. 审批、恢复、超时、取消。

这些东西一旦依赖隐式 loop 或全局 policy，后面做集成时就很容易互相污染。

第三层，`Runner` 为什么特别适合 Agent 宿主？

`asyncio.run()` 适合“一个程序，一个顶层 awaitable，然后退出”。这很适合简单 CLI。但 Agent 工程常常不是一次顶层调用就结束，而是：

1. 先初始化配置或凭据。
2. 再跑主会话。
3. 结束后再补 trace flush、审计落盘或清理动作。

如果这些顶层 async 步骤必须共享同一个 loop 和同一个 `contextvars.Context`，`Runner` 就更合适。因为它不是每次都新建一个 loop，而是在同一个宿主作用域里多次 `run()`。

这点对你这种从 Java / Spring 过来的人很好理解。它有点像你不想每个步骤都新起一个应用上下文，而是想在一个清晰的宿主生命周期里完成初始化、执行和清理。

第四层，`loop_factory` 为什么值得现在就记住？

因为它代表了官方推荐的新定制入口。以前很多人想换 loop 实现、做平台兼容或全局注入时，会第一反应用 policy。现在官方文档已经把方向改得很明确：**需要定制 loop 时，优先用 `loop_factory`。**

这背后的工程含义是：

1. loop 的来源应该跟宿主入口绑在一起。
2. 定制行为应该尽量局部化，而不是散落在全局状态里。
3. 后面升级 Python 版本时，迁移成本会更低。

如果你以后要做 MCP host、Agent worker 或长期运行的 CLI，这个原则比单纯记住某个事件循环类名更值钱。

第五层，为什么 `SIGINT` 处理值得单独拿出来说？

因为很多 Agent 程序不怕“报错退出”，怕的是“用户按了 Ctrl+C，但资源半清不清”。比如：

1. trace 还没 flush 完。
2. 子进程工具还没收尾。
3. MCP 连接还没正常关闭。
4. 本地缓存或 checkpoint 写到一半。

`Runner.run()` 的中断语义很像一种更工程化的退出流程：先取消 main task，让协程栈按 `CancelledError` 往上回卷，给 `finally` 清理块机会；之后再抛 `KeyboardInterrupt`。这和你熟悉的“先发停止信号，再做优雅停机”是同一个思路。

第六层，这和你现在的 Java / IoT 经验怎么桥接？

可以这样迁移理解：

1. `asyncio.run()` 像一个明确的应用主入口。
2. `Runner` 像一个更细粒度、可复用的宿主生命周期容器。
3. policy system 更像全局线程模型魔法，方便但容易积债。
4. `loop_factory` 则像把底层执行模型配置收回主入口，减少全局副作用。

这意味着，你在 Python Agent 工程里最该建立的，不是“会写很多 async 语法”，而是“会管理宿主生命周期”。

第七层，如果只记 4 条最落地规则，可以记这组：

1. 顶层程序默认用 `asyncio.run()`，不要再把“手动摸 loop”当成常态。
2. 需要多个顶层 async 步骤共享同一个 loop / context 时，用 `asyncio.Runner`。
3. 想定制 loop，优先用 `loop_factory`，不要继续把 policy 当长期方案。
4. 写 Agent CLI、MCP host、后台 worker 时，提前把 Ctrl+C 的取消和清理路径设计清楚。

这四条立住之后，你后面再学更复杂的 tool orchestration、MCP transport 或 eval runner，宿主层就不容易反复返工。

## 3 key takeaways

1. Python 官方已经明确把 `asyncio` policy system 标记为弃用，并计划在 Python 3.16 移除；Agent 宿主不该再依赖“全局 policy + 隐式 loop”的老路径。
2. `asyncio.run()` 适合单次顶层入口，`asyncio.Runner` 适合多个顶层 async 步骤共享同一个 event loop 和 `contextvars.Context`。
3. `Runner.run()` 的 `SIGINT` 处理体现了官方推荐的优雅退出模型：先取消 main task，让清理逻辑执行，再抛出 `KeyboardInterrupt`。

## relation to Agent engineering

这篇内容和 Agent engineering 的关系非常直接。

第一，它影响你怎么写宿主入口。以后无论是 Python CLI agent、MCP host、还是跑评测的 worker，入口怎么创建 loop、怎么持有 context、怎么退出，会直接决定代码是脚本味还是工程味。

第二，它影响你怎么做工具和上下文传播。Agent 系统往往需要把 `run_id`、trace、用户上下文挂在 `contextvars` 里；如果顶层宿主生命周期混乱，这些上下文很容易断掉或在错误边界里泄漏。

第三，它影响你怎么做清理和恢复。Browser session、subprocess tool、MCP transport、日志导出、trace flush，本质都依赖“宿主退出时能否按顺序取消和关闭”。`Runner` 的价值，正是在这层给出更稳定的官方骨架。

第四，它也在补你从 Java / Spring 到 Python 的关键迁移。你原来已经很熟悉“应用入口、线程模型、优雅停机、资源回收”这些工程概念；今天要做的，不是重新发明，而是把这套习惯迁到 `asyncio` 宿主上。

## a small action for tonight

今晚做一个 30 到 40 分钟的小实验，只做宿主，不做复杂业务：

1. 先写一个用 `asyncio.run()` 启动的最小脚本，在 `main()` 里打开一个后台任务，再用 `try/finally` 打印清理日志。
2. 运行后按一次 `Ctrl+C`，观察清理逻辑是否先执行，再结束程序。
3. 再把它改成 `with asyncio.Runner() as runner:`，分三次 `runner.run()`：初始化、主逻辑、flush 收尾，确认它们能共享同一个上下文变量。
4. 如果你手边有旧代码还在碰 `get_event_loop_policy()` 或自定义 policy，把它改成 `loop_factory` 骨架，哪怕先只返回默认 loop。
5. 最后记一条自己的迁移规范：以后 Agent 宿主代码里，优先检查“入口和退出是否清楚”，再检查业务逻辑。

## 原文关键段落翻译（人工翻译，放在文末）

1. `asyncio.run()` 会负责管理事件循环、结束异步生成器并关闭 executor；它应被用作 asyncio 程序的主入口，并且理想情况下只调用一次。
2. 如果要配置事件循环，推荐使用 `loop_factory`，而不是 policy system；`asyncio` policy system 已被弃用，并将在 Python 3.16 移除。
3. `asyncio.Runner` 是一个 context manager，用于让多个顶层 async 函数在同一个 event loop 和同一个 `contextvars.Context` 中运行。
4. `Runner.close()` 会结束异步生成器、关闭默认 executor、关闭 event loop，并释放内部保存的上下文。
5. `Runner.run()` 会在用户代码执行前安装自定义 `SIGINT` 处理器；第一次 `Ctrl+C` 会先取消 main task，让 `try/except`、`try/finally` 清理逻辑有机会运行，之后再抛出 `KeyboardInterrupt`。
6. `get_event_loop_policy()`、`set_event_loop_policy()`、`AbstractEventLoopPolicy`、`DefaultEventLoopPolicy` 以及 Windows 下的内置 policy 都已在 Python 3.14 起弃用，并计划在 Python 3.16 移除。
7. 在 Python 3.14 中，默认 policy 的 `get_event_loop()` 如果当前没有设置 event loop，会直接抛出 `RuntimeError`。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.python.org/3/library/asyncio-runner.html
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.python.org/3/library/asyncio-policy.html
