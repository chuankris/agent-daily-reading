# title

2026-06-19 为什么现在该补 `pytest-asyncio` 的 loop scope 和 `strict/auto` 模式

## original source

- 标题：Concepts
- 链接：https://pytest-asyncio.readthedocs.io/en/stable/concepts.html
- 来源类型：pytest-asyncio 官方文档
- 访问时间：2026-06-19

- 标题：Configuration
- 链接：https://pytest-asyncio.readthedocs.io/en/stable/reference/configuration.html
- 来源类型：pytest-asyncio 官方文档
- 访问时间：2026-06-19

## why read today

你这几天已经把 `pytest` 的参数化、日志断言、临时目录这些基础砖头补上了，但如果下一步要开始认真做 Agent 的 tool 封装、MCP server 适配、异步 I/O、并发调用和回归测试，迟早会撞上一个比“怎么写测试函数”更本质的问题：

**异步测试到底运行在哪个 event loop 里，测试框架又是按什么规则接管这些 async test / async fixture 的？**

这不是一个“语法细节”问题，而是异步 Agent 工程的运行边界问题。很多 Python 初学者刚开始写异步测试时，只知道给测试函数加一个 `@pytest.mark.asyncio`，但一旦项目里出现下面这些情况，测试就会开始变得很难解释：

- 同一个模块里，有的测试能共享状态，有的却不能
- 异步 fixture 明明能跑，但换个作用域后突然出问题
- 一个仓库里既有 `asyncio` 也有别的 async 库时，插件开始互相抢控制权
- 测试“看起来并发”，但实际是串行执行，性能和副作用判断都和直觉不一样

今天这两页官方文档正好在补这个缺口。对你这种做了 10 年 Java / Spring / IoT 集成与交付的人来说，它们最值得看的地方，不是“pytest 插件怎么配”，而是它们在帮你建立一套新的工程直觉：

- `event loop` 是异步测试的运行容器
- `loop_scope` 是异步测试隔离边界的一部分
- `strict/auto` 是测试框架对异步世界“接管到什么程度”的开关

这三件事一旦理解清楚，你后面写 Agent 回归时，很多“为什么这个 async fixture 污染了别的测试”“为什么这段工具调用在 CI 里不稳定”“为什么插件共存时行为变怪了”的问题，就不再是黑盒。

## original-text translation

`Concepts` 这一页先把 `pytest-asyncio` 的底层模型讲明白了：它不是随便给你一个 event loop 然后把所有 async 测试都丢进去跑，而是沿着 pytest 的 collector 层级来提供 event loop。也就是说，pytest 在收集测试时本来就有 `Session`、`Package`、`Module`、`Class`、`Function` 这些层级；`pytest-asyncio` 则为这些层级分别提供可对应的 event loop。默认情况下，每个异步测试运行在 `Function` 级别的 loop 上，也就是最细粒度、隔离性最高的那个 loop。

文档接着说明：如果若干测试共享同一个上层 collector，例如它们都在同一个 module 里，那么你可以通过 `@pytest.mark.asyncio(loop_scope="module")` 让这些测试运行在同一个 module 级别的 loop 上。官方还特别提醒，彼此相邻、逻辑上相关的测试最好使用相同的 loop scope；如果同一组相近测试混着使用不同作用域，会让测试代码更难读，也更难推断状态边界。

同一页里另一个很重要的点是 `strict` 和 `auto` 两种发现模式。`strict` 模式下，`pytest-asyncio` 只接管那些显式打了 `asyncio` marker 的测试函数，以及显式用 `@pytest_asyncio.fixture` 装饰的 async fixture。没有这些显式标记的 async 测试和 async fixture，它不会主动处理。这个设计是为了让 `pytest-asyncio` 能和其他异步测试插件和平共存，所以它也是默认模式。

而 `auto` 模式更激进一些：它会自动给所有异步测试函数加上 `asyncio` marker，也会接管所有异步 fixture，不管你用的是 `@pytest.fixture` 还是 `@pytest_asyncio.fixture`。官方给出的判断标准也很实用：如果你的项目只使用 `asyncio`，那么 `auto` 往往是更简单、更推荐的默认选择；如果你的仓库需要同时支持多种异步库，例如 `asyncio` 和 `trio`，那就应该更偏向 `strict`。

`Concepts` 页最后还强调了一件很容易被误解的事：`pytest-asyncio` 虽然是在跑异步测试，但它并不会默认把多个异步测试并发执行。它仍然像普通 pytest 一样按顺序执行测试，只是每个异步测试在自己的 event loop 上运行。文档特意举了两个 `await asyncio.sleep(2)` 的测试例子，说明总耗时大约是 4 秒而不是 2 秒。背后的理由不是“插件做不到并发”，而是为了保持测试隔离，避免测试之间互相干扰。

`Configuration` 这一页则把几个关键配置点说得更工程化。首先，`asyncio_mode` 可以通过配置文件或命令行设置，命令行优先级更高；如果你不显式配置，它默认就是 `strict`。其次，`asyncio_default_test_loop_scope` 控制异步测试默认使用什么 loop scope，默认是 `function`。再往前一步，`asyncio_default_fixture_loop_scope` 控制异步 fixture 默认使用什么 loop scope；目前如果这个配置未设置，它默认会跟随 fixture 自身的 scope，但文档明确写了：**未来版本里，未设置时会默认变成 `function`。**

这个变化看起来只是一个默认值提醒，实际上很值得今天就记住。因为如果你以后在 Agent 项目里大量依赖 module/session 级异步 fixture，而又一直靠“默认行为”在跑，那么插件升级后，测试隔离边界可能会悄悄变化，带来一些表面像偶发 bug、实则是 loop scope 漂移的问题。

`Configuration` 还提到 `asyncio_debug`：你可以在配置里打开 asyncio debug mode，也可以通过命令行参数打开。对 Agent 工程来说，这在排查悬空任务、未正确 await、事件循环生命周期异常时很有价值，尤其适合配合最近你在补的日志与可观测性意识一起用。

## Chinese deep summary

如果把今天这两页压缩成一句工程判断，就是：

**异步 Agent 测试的核心不只是“能跑”，而是你要主动决定测试框架如何接管 async 代码，以及这些 async 测试共享还是隔离 event loop。**

第一层，你要把 `event loop` 理解成异步测试的“运行容器”，而不只是 asyncio 的背景板。

很多刚从 Java 转到 Python 的工程师，会把异步测试理解成“线程版单元测试的另一个语法壳”。这种理解不够，因为 Python 的异步测试能否稳定，常常不是由断言语句决定，而是由 event loop 的边界决定。  
在你过去的 Java / IoT 集成经验里，你其实早就熟悉这种问题，只是名字不一样。比如：

- 一个集成测试跑在独立容器里，还是共用同一个应用上下文？
- 一组测试共享连接池、共享嵌入式 broker、共享 mock server，会不会互相污染？
- 某个资源是“每次请求新建”，还是“整个模块复用”？

在 `pytest-asyncio` 里，`loop_scope` 就是在回答类似问题。`function` scope 更隔离，更适合保证每条 Agent 回归互不影响；`module` 或 `session` scope 则更像共享运行容器，适合那些明确要复用昂贵异步资源的场景。

第二层，`strict` 和 `auto` 的区别，本质上是在决定“框架要不要替你做更多推断”。

对 Python 新手来说，`auto` 很诱人，因为省心。项目只要是纯 `asyncio` 技术栈，它往往确实更顺手：异步测试不用反复显式打 marker，async fixture 也更不容易因为装饰器选错而悄悄失效。  
但从工程治理角度看，`strict` 更保守，它要求你显式声明“这段测试归 asyncio 管”。这种方式在多插件共存、多异步模型共存的仓库里更稳，因为它减少了插件之间互相猜测、互相抢活的空间。

这和你以前做系统集成时的经验很像：

- `auto` 更像“平台帮你自动装配”
- `strict` 更像“关键边界必须显式声明”

两者没有绝对高下，关键是仓库形态不同。如果你的 Agent 仓库就是单一 `asyncio` 路线，`auto` 往往能降低样板代码；如果它要接更多实验性异步栈，或者团队成员很多、代码风格不统一，`strict` 的边界感会更有价值。

第三层，异步测试默认串行执行，这一点对 Agent 回归尤其重要。

很多人看到 `async` 就会默认联想到“并发更快”。但 `pytest-asyncio` 官方明确强调，测试仍是顺序执行的。原因很现实：测试的首要目标不是榨干吞吐，而是保证隔离、可重复、可定位。  
这对 Agent 工程的启发很直接：当你写 tool calling、MCP server、异步 HTTP 调用、文件导出或缓存命中回归时，不要把“函数里用了 async/await”误当成“整个测试套件会自动并发”。如果你未来需要性能压测或并发行为验证，那通常是另一层测试目标，不能和基础回归混在一起。

第四层，`asyncio_default_fixture_loop_scope` 的未来默认值变化，提醒你别把关键边界寄托在隐式默认行为上。

这一点对转型中的你非常关键。因为你现在还在快速积累 Python 手感，最容易形成的坏习惯就是“先让它跑起来，配置以后再说”。但异步测试里，一旦 loop scope 这种边界靠默认值兜底，后面版本升级、插件升级、团队协作接入时，问题往往不是立即报错，而是变成：

- 某些测试开始偶发失败
- 某些共享状态突然不再共享
- 某些异步 fixture 生命周期变短了
- 某些清理逻辑在本地和 CI 表现不一致

这些问题最难，因为它们不是语法错误，而是运行语义漂移。官方文档已经提前告诉你未来默认值可能收紧到 `function`，那更稳的做法就是：**凡是你真的依赖的 loop scope，都显式写出来，不要赌默认值永远不变。**

第五层，把今天内容放回 Agent engineering，你会发现它补的是“异步测试运行模型”这块地基。

你最近读的 `caplog`、`tmp_path`，解决的是日志证据和文件副作用；更早读的 `asyncio` 系列，解决的是任务取消、超时、线程边界和调度语义。但要把这些东西真正串起来，还需要知道：**pytest 到底怎么承载这些异步行为。**  
`pytest-asyncio` 的 loop scope 和模式选择，就是这层承载逻辑。没有这层理解，你后面即使会写 async 工具、会写回归，也很容易把不稳定误以为是“模型随机性”，其实很多时候是测试容器边界没设计清楚。

## 3 key takeaways

1. `pytest-asyncio` 不是简单“让 async 测试能跑”，而是按 pytest collector 层级提供 event loop；默认 `function` scope 代表最高隔离性。
2. `strict` 适合多异步库或多插件共存的仓库，`auto` 适合纯 `asyncio` 项目；这本质上是在选择“显式边界优先”还是“降低样板优先”。
3. 异步测试默认仍然串行执行，而且异步 fixture 的默认 loop scope 行为未来可能收紧；关键 loop scope 应显式配置，不要依赖隐式默认值。

## relation to Agent engineering

这和 Agent engineering 的关系非常直接。

第一，它决定你怎么搭 Agent 回归环境。只要你的 Agent 涉及异步 HTTP 请求、异步工具、异步队列、异步 MCP server 或异步 tracing，上层测试框架的 loop 作用域就会影响状态隔离、资源复用和故障定位。

第二，它会改变你设计 fixture 的方式。你以后写异步测试资源时，应该开始主动区分哪些资源必须函数级隔离，哪些资源可以模块级复用，哪些资源需要显式声明 loop scope 才不会在升级后悄悄变语义。

第三，它非常适合你的背景迁移。你原来在 Java / Spring 里擅长的是系统边界、资源生命周期、环境隔离和交付稳定性；今天要补的不是“为什么要管边界”，而是用 Python / pytest-asyncio 的语言，把这些边界重新表达出来。

## a small action for tonight

今晚做一个 45 分钟以内的小动作，把今天的概念转成你自己的测试直觉：

1. 在本地随手建一个最小测试文件，写 2 个 `async` 测试和 1 个异步 fixture。
2. 先用 `strict` 模式跑一次，分别试试“有 marker / 无 marker”“`@pytest.fixture` / `@pytest_asyncio.fixture`”四种组合，观察哪些会被接管、哪些不会。
3. 再切到 `auto` 模式跑一次，感受它替你自动接管后的差异。
4. 最后把其中两个测试改成 `loop_scope="module"`，用一条共享状态断言确认它们确实跑在同一个 loop 上。
5. 顺手在 `pyproject.toml` 里显式写下你想要的 `asyncio_mode`，并记一条规则：以后只要仓库依赖异步 fixture，就显式检查 loop scope，不让默认值替你做关键决定。

## 原文关键段落翻译（人工翻译，放在文末）

1. `pytest-asyncio` 会为每一个 pytest collector 提供一个 asyncio event loop；默认情况下，每个测试使用 `Function` collector 提供的 loop，也就是最窄作用域、隔离性最高的 loop。
2. 如果多个测试共享同一个上层 collector，可以通过 `loop_scope` 参数把它们安排到该上层 collector 对应的同一个 loop 中运行；相邻测试最好保持一致的 loop scope，混用不同作用域会降低可读性。
3. `strict` 模式只处理显式标记为 asyncio 的测试和显式用 `@pytest_asyncio.fixture` 声明的异步 fixture，这样更适合与其他异步测试插件共存，因此它是默认模式。
4. `auto` 模式会自动接管所有异步测试函数和所有异步 fixture；如果项目只使用 `asyncio`，这是更简单也更推荐的默认选择。
5. `pytest-asyncio` 会顺序执行异步测试，而不是默认并发执行；这样做是为了维持测试隔离，避免测试之间产生竞争条件和副作用污染。
6. `asyncio_default_fixture_loop_scope` 决定异步 fixture 默认使用的 loop scope；当前未设置时会跟随 fixture 自身的 scope，但未来版本里未设置时将默认变成 `function`。
7. `asyncio_mode` 可以在配置文件或命令行中设置；如果两边都设置了，命令行优先；如果都没设置，默认值是 `strict`。
8. `asyncio_debug` 可以为异步测试和异步 fixture 使用的默认 event loop 打开 asyncio debug mode，这对排查事件循环相关问题有帮助。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://pytest-asyncio.readthedocs.io/en/stable/concepts.html
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://pytest-asyncio.readthedocs.io/en/stable/reference/configuration.html
