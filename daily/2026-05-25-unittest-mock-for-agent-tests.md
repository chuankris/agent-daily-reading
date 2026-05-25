# title

2026-05-25 为什么 `unittest.mock` 是 Python Agent 工程测试的第一道护栏

## original source

- 标题：`unittest.mock — mock object library`
- 链接：https://docs.python.org/3/library/unittest.mock.html
- 来源类型：Python 官方文档
- 访问时间：2026-05-25

- 标题：`unittest.mock — getting started`
- 链接：https://docs.python.org/3.14/library/unittest.mock-examples.html
- 来源类型：Python 官方示例文档
- 访问时间：2026-05-25

## why read today

你这几天已经连续补了 `inspect.signature()`、`Annotated`、JSON Schema、OpenAPI 这条线，已经能看懂“一个 Python 函数怎样变成 Agent tool”。但真实工程里，下一步不是继续堆概念，而是要解决一个更硬的问题：

“当工具函数里有 HTTP 调用、MCP client、文件 IO、异步会话、上下文注入时，我怎么在不真的打外部依赖的情况下，把行为稳定测出来？”

这正是 `unittest.mock` 的位置。

对你这种做过很多年 Java/Spring/IoT 集成的人，这个问题并不陌生。你以前会用 Mockito、Stub Server、接口模拟、依赖注入替身来隔离外部系统。Python 这边也一样，只是语法和约束不一样：

1. Python 里常见手法不是接口先行，而是直接 patch 名字绑定。
2. 异步函数不是普通 `Mock`，而是 `AsyncMock`。
3. 如果不用 `autospec`，很多 mock 会“假得太宽松”，测试能过，但生产代码参数早就对不上了。
4. 如果 patch 的位置错了，测试也可能“看起来 patch 了，实际上没 patch 到”。

对 Agent engineering 来说，这些不是测试细节，而是系统可靠性的底层护栏。因为一个 Agent 宿主最容易坏的地方，恰恰不是 prompt，而是工具调用、上下文注入、网络边界、重试逻辑和异常分支。

## original-text translation

Python 官方文档先给了 `unittest.mock` 一个很工程化的定义：它是用来在测试中替换被测系统某些部分的库，替换之后，你可以断言这些对象被怎样调用过、传入了什么参数、返回了什么结果。它的核心心智不是“录制再回放”，而是“动作发生后做断言”。

文档随后强调了 `patch()` 的本质：临时把某个名字指向的对象替换掉，并在测试结束后恢复。关键不在“原对象定义在哪里”，而在“被测代码实际从哪里查找这个名字”。也就是说，patch 的原则不是 patch 定义点，而是 patch lookup 点。

官方文档对 `autospec` 的解释非常重要。`autospec` 会让 mock 尽量保留原对象的 API 形状和调用签名。这样一来，如果真实函数需要三个参数，而测试代码错误地只传了一个参数，mock 也会像真实函数一样抛出 `TypeError`，而不是因为 mock 太宽松而把错误吞掉。

对异步代码，文档说明了 `AsyncMock` 的语义：它看起来像一个 async function，调用结果是可 await 的；只有真正 `await` 过之后，相关断言才成立。也就是说，“被调用过”与“被 await 过”在 Python 异步测试里是两个不同层级的事实，不能混为一谈。

官方示例页进一步把一个常见坑讲得很清楚：如果你 patch 一个类上的未绑定方法，又希望断言是谁调用了它，那么直接用普通 mock 往往拿不到 `self`。这时 `autospec=True` 会用一个保留真实签名的函数壳去代理底层 mock，于是这个方法从实例取出时仍然会正确绑定 `self`。

示例页还提醒了另一个实战问题：如果在 `setUp` 里手动 `start()` patcher，但异常导致 `tearDown` 没有执行，patch 可能泄漏到后续测试。官方建议用 `addCleanup()` 来兜底，保证 patch 能被回收。

## Chinese deep summary

如果把今天这篇材料压成一句话，那就是：`unittest.mock` 不是“随手造个假对象”，而是 Python Agent 工程里用来控制依赖边界、维持测试真实性、验证异步行为的基础设施。

第一层，你要把它看成“依赖隔离工具”，不是“语法糖”。

很多刚转 Python 的工程师会把 mock 理解成“把返回值改一下”。这太浅了。对 Agent 宿主来说，mock 真正解决的是边界控制问题。比如：

1. tool 函数内部会不会真的发 HTTP 请求。
2. MCP client 有没有真的连外部 server。
3. tracing/exporter 有没有真的往外发数据。
4. 文件系统、环境变量、时间函数会不会把测试跑成不稳定结果。

你以前做 ToB 交付时，最讨厌的是联调环境不稳定，导致一个本来能在本地验证的逻辑，被迫卡在外部依赖上。Python 里的 mock，本质上就是把“联调依赖”和“本地行为验证”拆开。先测宿主逻辑是否正确，再测与外部系统的集成是否正确。

第二层，`patch` 不是改对象，而是改“名字绑定”。

这是 Java 背景的人最容易别扭的地方。Java/Mockito 的常见心智是“替换某个依赖对象”；Python `patch()` 的常见心智则是“在某个命名空间里，把这个名字暂时指向另一个对象”。所以真正重要的问题变成了：

“被测代码运行时，是从哪个模块名字空间拿到这个对象的？”

如果你 patch 错模块，测试不会报很明显的错，但行为根本没被替换，这就是最危险的假绿测试。官方文档反复强调 “patch where it is looked up”，这句话对 Agent 工程特别重要，因为 Agent 宿主经常会跨模块封装：

1. `tool_runner.py` 引用了 `clients.py` 里的 client。
2. `session_service.py` 引用了 `datetime`、配置对象、trace helper。
3. `mcp_gateway.py` 引用了 transport、serializer、auth provider。

这些地方如果 patch lookup 点不准，你以为测的是隔离后的宿主逻辑，实际测的还是半真半假的外部调用。

第三层，`autospec` 是防止 mock 过度宽松的关键开关。

Python 的普通 `Mock` 很灵活，灵活到有时反而危险。因为它会接受很多本不该接受的调用方式。对 demo 来说这很方便，但对工程测试来说，这会让你的测试失去约束力。

`autospec` 的价值，不在于“更高级”，而在于“更像真的”。它至少帮你守住三件事：

1. 方法名和属性名不能乱写。
2. 调用签名尽量跟真实对象一致。
3. patch 类方法、未绑定方法时，绑定行为更接近真实运行时。

这对你当前阶段尤其关键。你正在补 Python，不像长期 Python 工程师那样对函数签名、bound/unbound method、descriptor 行为有肌肉记忆。此时如果大量使用不带 `autospec` 的 mock，测试会让你形成错误反馈：代码其实已经偏了，但测试仍然一片绿。

第四层，异步 Agent 代码必须区分“call”与“await”。

这是今天最值得你刻意建立的 Python 直觉。官方文档明确说明：`AsyncMock` 被调用，并不等于它已经被 `await`。只有真正 await 了，`assert_awaited*` 这套断言才成立。

这在 Agent 工程里太常见了。很多宿主 bug 不是“没调用”，而是：

1. coroutine 创建了，但没 await。
2. 重试逻辑里漏 await 了一层包装。
3. 并发收集任务时只保存 coroutine，没有提交或等待。
4. tool 执行链提早返回，导致真正的异步副作用没发生。

如果你只用 `assert_called_once_with()` 这类同步心智去测异步函数，很容易漏掉最关键的执行事实。`AsyncMock` 的断言 API 本质上是在逼你把“调度发生了”和“执行落地了”区分开。

第五层，mock 的意义不是代替集成测试，而是让你把测试金字塔重新立起来。

你过去做系统集成，应该很熟悉一个现实：全靠联调验收，交付成本一定高。Agent 工程也是一样。好的结构不是“所有逻辑都跑真模型、真工具、真外部服务”，而是分层：

1. 单元层：用 mock 隔离外部依赖，验证控制流、异常分支、重试、状态传递。
2. 契约层：验证 tool schema、MCP 请求/响应结构、JSON 数据边界。
3. 集成层：少量跑真 client、真沙箱、真模型。
4. 评测层：从任务完成率、稳定性、成本和可追踪性看系统表现。

你现在补 `unittest.mock`，不是为了学一个测试模块，而是在补“怎样把 Agent 宿主从一次性脚本提升到可持续迭代的工程系统”这块基本功。

## 3 key takeaways

1. `patch()` 的核心原则不是 patch 定义位置，而是 patch 被测代码实际查找对象的命名空间；patch 错位置，最容易产生假绿测试。
2. `autospec=True` 不只是“更严格一点”，而是让 mock 保留真实 API 和调用签名，避免 Python 动态性把参数错误、方法错误悄悄吞掉。
3. 对异步 Agent 代码，`AsyncMock` 和 `assert_awaited*` 必须成为默认心智；“被调用”不等于“被 await 并真正执行”。

## relation to Agent engineering

这篇内容和 Agent engineering 的关系非常直接，而且几乎覆盖你当前最该补的“Python 工程化 + Eval/可靠性”交叉区。

第一，测 tool wrapper。你写的很多 tool 不会直接做业务，而是包装 HTTP client、MCP session、数据库、文件系统或子进程。这里最需要验证的是参数传递、超时、异常翻译、重试和日志埋点，而不该每次都真的打外部系统。`mock` 正好把这层隔离开。

第二，测宿主编排。一个 Agent run 往往要做 “取上下文 -> 选工具 -> 调工具 -> 处理失败 -> 写 trace -> 返回结果”。这类代码不是算法难，而是边界多、分支多、异步多。mock 能让你把每一跳副作用都控制住，只测编排本身。

第三，测 MCP 和外部协议桥接。你从 Java/IoT 转过来，对协议、契约、网关、适配层并不陌生。MCP host/client 其实也很像一套新的集成边界。你需要的不是“会调一次”，而是能稳定验证 transport 层失败、授权失败、工具超时、资源读取异常这些分支。mock 是你在 Python 里补这套能力的最短路径。

第四，支撑 eval。很多人把 eval 只理解成数据集打分，但如果底层宿主没有可控测试手段，eval 结果也会掺进大量环境噪声。先把 `mock`、契约测试、少量集成测试立住，再做 Agent eval，结果才有可解释性。

第五，帮助你把 Java 经验迁移过来。你不缺“工程边界意识”，缺的是 Python 世界里对应的表达方式。把 `patch` 看成名字绑定替换，把 `autospec` 看成更接近 Mockito 严格桩，把 `AsyncMock` 看成专门面向 coroutine 的替身，你会更快把已有经验迁移进来。

## a small action for tonight

今晚做一个 35 分钟的小练习，不求完整，只求把“异步工具测试”手感建立起来：

1. 写一个 `tool_runner.py`，里面有 `async def run_tool(client, query):`，函数内部 `await client.search(query)`，失败时把异常转成你自己的错误类型。
2. 写一个测试文件，用 `AsyncMock()` 模拟 `client.search`，分别覆盖成功、抛异常、被调用但未 await 三种场景。
3. 在成功场景里同时写 `assert_called_once_with()` 和 `assert_awaited_once_with()`，亲手感受两者语义不同。
4. 再把 `client.search` 的 patch 改成 `autospec=True` 的方式，故意传错参数，观察测试如何从“错误被吞掉”变成“立即报签名错误”。
5. 最后故意 patch 错模块路径一次，验证“为什么测试没拦住真实调用”，把这个坑记牢。

如果你今晚只做这一轮，后面再看 FastAPI 依赖注入测试、MCP client 测试、Agent run 编排测试时，理解速度会明显快很多。

## 原文关键段落翻译（人工翻译，放在文末）

1. `unittest.mock` 是一个用于测试的库。它允许你把被测系统的部分组件替换成 mock 对象，并对这些对象被如何使用进行断言。
2. `patch()` 的关键原则是：你要 patch 的是被测系统查找该对象时使用的名字，而不一定是这个对象最初被定义的地方。
3. 自动加规格（autospeccing）会创建与被替换对象拥有相同属性、方法和调用签名的 mock；如果使用方式不对，它会像真实代码一样失败。
4. 如果 patch 的目标是异步函数，那么 `patch()` 会返回 `AsyncMock`；同样，`create_autospec()` 针对异步函数也会返回 `AsyncMock`。
5. `AsyncMock` 被调用后返回的是一个可等待对象；只有真的执行了 `await`，相关的 awaited 断言才会成立。
6. 对未绑定方法使用 `patch.object(..., autospec=True)` 时，patch 出来的函数壳会保留原始签名，并在实例访问时正确绑定 `self`。
7. 如果你在 `setUp` 里手动启动 patcher，要确保后续一定能 `stop()`；`addCleanup()` 可以在异常情况下也保证 patch 被回收。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.python.org/3/library/unittest.mock.html
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.python.org/3.14/library/unittest.mock-examples.html
