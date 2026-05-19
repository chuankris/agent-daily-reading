# title

2026-05-16 为什么 `ExceptionGroup` 和 `except*` 是你读懂 Python Agent 并发失败语义的关键一课

## original source

- 标题：Built-in Exceptions - Python 3.14.5 documentation
- 链接：https://docs.python.org/3/library/exceptions.html#exception-groups
- 来源类型：官方文档
- 访问时间：2026-05-16

- 标题：PEP 654 - Exception Groups and except*
- 链接：https://peps.python.org/pep-0654/
- 来源类型：官方规范 / PEP
- 访问时间：2026-05-16

## why read today

你这几天已经连续补了 `TypedDict`、`TypeAdapter`、`JSON Schema`、`Protocol`，也在更早的时候看过 `TaskGroup` 和取消语义。现在差的不是“再多会几个 Python 语法点”，而是把一个真实 Agent 宿主最容易出事故的失败路径补上：

1. 并发拉多个工具结果时，不是只会失败一个分支。
2. `TaskGroup` 在多个子任务失败时，不会只给你一个异常，而是聚合成一棵异常树。
3. 如果你还按旧的 `try/except` 心智写代码，很容易只处理了自己关心的那一类错误，同时把别的错误一起吞掉。

这对你这种 10 年 Java / Spring / IoT 集成 / ToB 交付背景的人很关键。你原来很熟悉“主流程失败时，多个下游分支一起回收、一起汇总、一起留痕”的工程语义；Python 3.11+ 只是把这套语义用语言级能力正式表达出来了。今天这篇不是学一个新花样，而是把你后面做 Agent orchestration、MCP client、并发评测、trace flush、资源清理时会遇到的错误模型补完整。

## original-text translation

Python 官方在异常文档里把 `ExceptionGroup` / `BaseExceptionGroup` 定义得很直接：当程序需要一起抛出多个彼此无关的异常时，就使用异常组。`ExceptionGroup` 属于 `Exception` 体系，因此普通的 `except Exception` 可以捕获它；而 `BaseExceptionGroup` 可以包装更宽的 `BaseException` 子类，所以它不应该被普通的 `except Exception` 误抓进去。官方还强调，异常组不是“把异常随便塞进列表”这么简单，它保留了分组、嵌套、`__traceback__`、`__cause__`、`__context__` 等调试信息，并提供 `subgroup()` / `split()` 这样的拆分能力。

PEP 654 进一步解释了为什么必须引入新的 `except*`，而不是偷偷改掉旧 `except` 的语义。原因是：异常组里可能同时包含你想处理和你不想处理的错误，如果继续沿用传统 `except`，很容易把不该吞掉的错误一起吃掉。`except*` 的设计就是为了按类型把异常组拆成子组，匹配到的部分进入当前处理分支，没匹配到的部分继续向后传递或重新抛出。PEP 还明确指出，一旦一个库从“抛单个异常”改成“抛异常组”，这通常是 API 级别的变化，调用方也要从 `except` 心智升级到 `except*` 心智。

换句话说，官方想表达的是：并发失败不是“多条日志”这么简单，而是一个需要被结构化表示、精确拆分、避免误吞的异常传播问题。

## Chinese deep summary

如果把今天这组材料压成一句话，那就是：`ExceptionGroup` 解决的是“多个失败如何一起向上传播”，`except*` 解决的是“调用方如何只处理自己该处理的那部分失败，同时不破坏其余失败的可见性”。

这件事为什么对 Agent 工程特别重要？因为 Agent 系统比传统 CRUD 后端更容易出现“并发但不独立”的失败。比如一个请求里：

1. 同时打两个搜索源。
2. 并发调两个 MCP 工具。
3. 一边等模型输出，一边做 usage 统计、trace 写出、会话状态落盘。
4. 在总超时到来时，同时回收尚未完成的子任务。

这些子任务不是随便并发一下而已，它们共同组成了一次用户请求。当其中一个分支失败时，剩余分支是否取消、最终对外抛出一个还是多个错误、调用方如何分门别类处理，都会直接影响系统可观测性和恢复策略。

你如果还停留在“异常总是单个对象”的心智里，后面会遇到两个典型坑。

第一个坑，是把异常组当成普通异常去 `except SomeError`。这在一些 API 边界上可能根本抓不到你想抓的内容，因为异常已经不是“一个 `ValueError`”，而是“一个包含多个子异常的组”。这时真正需要的是 `except* ValueError` 这种“从组里抽子组”的处理方式。

第二个坑，是把异常组一把抓住之后只盯着自己关心的那部分，然后把剩余异常不小心吞掉。PEP 654 专门强调了这个风险，所以才新增 `except*` 而不是改造旧 `except`。对工程系统来说，吞错比报错更危险，因为它会制造“表面成功、内部半失败”的脏状态。

从 Java 背景迁移过来，可以把它类比成这样：过去你在 `CompletableFuture.allOf(...)`、批量下游调用、批处理汇总里，真正关心的不只是“失败了”，而是“哪些子调用失败了，哪些是同类故障，哪些要重试，哪些要立即上抛”。Python 3.11+ 不是让你放弃这种工程视角，而是把它语言内建化了。

还有一个非常实用的点：异常组本身是“树”，不是“平铺列表”。这意味着它适合表示嵌套并发结构。比如外层是“一个用户请求”，里面有“检索阶段”和“工具阶段”，每个阶段里又有多个并发子任务。异常树比一堆打平日志更适合后续做 trace 关联、失败归因、重试分层和用户可解释错误输出。

另外，今天这篇也能帮你更准确理解前面看过的 `TaskGroup`。`TaskGroup` 的价值不只是帮你取消其余任务，更重要的是它最终会把多个非取消异常聚合后抛出。这说明结构化并发的“收口”并不是简单地选一个报错代表全体，而是尽可能保留并发失败的完整信息。你后面写 Agent host、batch eval runner、并发工具执行器时，异常边界就应该按这个标准设计。

最后要记住一个 API 设计层面的判断：PEP 654 明说，“从抛单个异常改成抛异常组”通常算 API breaking change。对你以后自己封装 Agent 运行时接口也一样。如果一个函数可能抛出异常组，这件事必须在接口契约、测试、调用方处理方式里写清楚，不能让调用方按单异常假设去猜。

## 3 key takeaways

1. `ExceptionGroup` 表达的是“多个彼此无关的异常一起传播”，它保留嵌套结构和调试元数据，不是简单的异常列表。
2. `except*` 的职责不是替代普通 `except`，而是在异常组场景下按类型拆分子组，处理一部分并让剩余部分继续传播，避免误吞错误。
3. 对 Agent 工程来说，并发失败语义是接口契约的一部分；谁会聚合异常、谁负责拆分、谁允许部分处理，都应该被明确设计和测试。

## relation to Agent engineering

这篇内容和 Agent 工程的关系非常直接。

第一层关系是并发工具调用。你后面无论是自己写 tool router，还是套 OpenAI / MCP / 自定义 provider，都很容易出现“一次请求里多个分支一起失败”的情况。没有异常组心智，你会把这些失败压扁成单个异常，导致排障信息严重丢失。

第二层关系是 Host 边界设计。一个成熟的 Agent host 不该只是“能跑通”，还要定义清楚：并发子任务失败时，是立即取消同组任务，还是允许部分完成；最终是抛单异常、异常组，还是做领域级包装。今天这篇正是这条边界的基础。

第三层关系是 Eval / Trace / Observability。评测批跑、回放、日志归因、失败聚类，本质上都依赖你能不能保留“这次失败里其实有多个叶子异常”的信息。异常组天然比“取第一个异常 message 记日志”更适合这些工作。

第四层关系是你自己的迁移路径。你并不缺系统集成经验，缺的是把原来对失败传播、超时回收、分支归因的判断，翻译成 Python 3.11+ 的语言模型。`ExceptionGroup` / `except*` 正是这层翻译的关键接口。

## a small action for tonight

今晚做一个 30 分钟的小实验，不追求业务功能，只练失败收口：

1. 写一个 `asyncio.TaskGroup` 小脚本，并发启动 3 个任务，其中 2 个分别抛 `ValueError` 和 `TypeError`。
2. 第一版先用普通 `try/except Exception` 包外层，观察你拿到的到底是什么对象、打印出来长什么样。
3. 第二版改成 `except* ValueError` 和 `except* TypeError`，观察两个分支是否都会执行，以及剩余未匹配异常如何继续传播。
4. 再加一层你自己的领域包装，例如把“工具阶段失败”包成一层更高阶异常组，感受异常树和打平列表在可读性上的差异。
5. 最后写一句总结：以后在你的 Agent 代码里，哪些地方应该只抛单异常，哪些地方允许抛异常组。

## 原文关键段落翻译（人工翻译，放在文末）

1. 当确实需要把多个彼此无关的异常一起向上传播时，Python 提供 `ExceptionGroup` 和 `BaseExceptionGroup` 这两类异常来承载这件事。
2. `BaseExceptionGroup` 可以包装任意 `BaseException`；`ExceptionGroup` 只能包装 `Exception` 子类，这样普通的 `except Exception` 会抓到前者中的安全子集，而不会误把 `KeyboardInterrupt` 之类也吞进去。
3. `subgroup()` 和 `split()` 的意义，是在保留原有嵌套结构、traceback、cause、context 等调试信息的前提下，把异常组按条件拆成更小的组。
4. PEP 654 引入 `except*`，是因为并发失败里常常既有你想处理的错误，也有你不该吞掉的错误；新语法允许你只处理匹配子组，其余部分继续传播。
5. 对库作者来说，把一个 API 从“抛单异常”改成“可能抛异常组”，通常应视为接口行为变化，调用方需要相应升级处理方式。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.python.org/3/library/exceptions.html%23exception-groups
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://peps.python.org/pep-0654/
