# title

2026-05-23 为什么 `inspect.signature()` / PEP 362 是你理解 Agent 工具契约的底层入口

## original source

- 标题：`inspect — Inspect live objects`（Python 3.13.13 documentation）
- 链接：https://docs.python.org/3.13/library/inspect.html
- 来源类型：Python 官方文档
- 访问时间：2026-05-23

- 标题：PEP 362 – Function Signature Object
- 链接：https://peps.python.org/pep-0362/
- 来源类型：Python 官方 PEP / 标准设计说明
- 访问时间：2026-05-23

## why read today

你这几周已经连续补了 `Pydantic`、`TypedDict`、`JSON Schema`、`OpenAPI`、`MCP tool contract`。这些内容解决的是“契约长什么样”。但只要你开始自己写 Agent 宿主、tool registry、函数适配层，马上会遇到更底层的问题：

1. Python 框架到底怎么从一个函数里读出参数名、默认值、参数种类和返回注解？
2. 为什么有些装饰器包了一层之后，工具签名还能保住；有些却会丢？
3. 为什么同样是“把参数传给函数”，成熟框架不会自己手搓 `args/kwargs`，而是先做一次绑定检查？

今天这组材料补的不是“再学一个库”，而是你以后读 LangChain、OpenAI Agents SDK、MCP server/host、FastAPI、Typer 这类框架源码时反复会撞见的底层机制。

对你这种做过 10 年 Java / Spring / IoT 集成的人，这个主题尤其值钱。因为它本质上就是 Python 世界里的“反射 + 方法签名 + 参数绑定规则”，只是表达方式不是 `Method` / `Parameter[]`，而是 `Signature` / `Parameter` / `BoundArguments`。你一旦把这层看明白，Python Agent 工程里很多“自动注册工具”“自动生成 schema”“自动做调用校验”的神秘感就没了。

## original-text translation

Python 官方文档先给出一个很关键的事实：`Signature` 对象表示“一个可调用对象的调用签名以及返回注解”，要拿到它，应该用 `inspect.signature()`。这个 API 能处理的对象范围很广，不只是普通函数，还包括类、`functools.partial()` 之类的可调用对象。

文档接着强调了几个对工程实现非常重要的边界：

1. 函数签名里如果出现 `/`，表示它前面的参数是 positional-only，只能按位置传，不能按关键字传。
2. `signature()` 默认会沿着 `__wrapped__` 去展开装饰器；如果你不想展开，可以把 `follow_wrapped=False`。
3. 某些对象在某些 Python 实现里可能无法被完整内省，比如一部分 C 实现的内建函数并不提供足够的参数元数据。
4. 如果对象带有 `__signature__`，CPython 可能会用它来构造签名，但这部分语义属于实现细节，不能想当然地依赖。

PEP 362 则把这套机制讲得更“工程设计”一些。它提出：不要再让程序员去翻函数对象上一堆分散的底层属性，而是提供统一的 `Signature` / `Parameter` / `BoundArguments` 表示。`Signature.bind(*args, **kwargs)` 的作用，是先按 Python 真正的调用规则把实参和形参做一次映射；如果不匹配，直接抛 `TypeError`。这其实就是把“参数校验”从业务逻辑里提前抽出来。

PEP 还说明了 `signature()` 的大致工作顺序：先看对象是否可调用，再优先看 `__signature__`，然后考虑 `__wrapped__`、普通函数、绑定方法、`functools.partial`、类的 `__call__` / `__new__` / `__init__` 等情况。也就是说，Python 生态里很多“自动识别函数签名”的能力，并不是只对最普通的裸函数有效，而是对一整类可调用对象都尽量统一处理。

最后一个很容易被忽略、但对 Agent 宿主很重要的点是：PEP 362 明确说 `Signature` 是惰性创建的，而且**不会隐式缓存**。如果你需要缓存，可以手动放进 `__signature__`。这意味着框架作者必须自己想清楚缓存策略，而不能默认以为 introspection 结果天然会帮你记住。

## Chinese deep summary

如果把今天的内容压成一句话，那就是：`inspect.signature()` 让 Python 的“可调用契约”第一次变成了可编程的一等对象，而 PEP 362 规定了这件事应该怎样被统一表达。

为什么这对 Agent 工程重要？因为 Agent 系统的很多自动化能力，本质都建立在“函数契约可被读取、可被校验、可被改写”之上。

第一层是“读契约”。

一个工具函数不是只有名字。对 Agent 宿主来说，至少还要知道：

1. 哪些参数必填，哪些有默认值。
2. 哪些参数只能 keyword-only，哪些可以 positional。
3. 有没有 `*args` / `**kwargs` 这种开放边界。
4. 参数注解和返回注解是什么。

`Signature.parameters` 和 `Parameter.kind/default/annotation` 正是在做这件事。你以后看工具注册代码时，如果它先 `signature(func)`，再遍历 `sig.parameters.values()`，不要觉得这是“Python 小技巧”；那其实是在抽取工具契约。

第二层是“校验调用”。

很多人初学 Python 动态调用，会直接自己组 `kwargs` 然后 `func(**kwargs)`。问题是，这么写虽然快，但错误出现得太晚，而且错误信息往往和你的宿主上下文脱节。

`Signature.bind()` 的价值，是先把“这个调用是否合法”独立出来做一遍。它会按 Python 的真实规则检查：

1. 必填参数有没有缺。
2. 同一个参数有没有被重复传。
3. positional-only 参数是不是被错误地按关键字传了。
4. keyword-only 参数是不是被遗漏或错误地塞进位置参数。

这和你以前在 Java/Spring 集成里强调“参数装配先校验，再进业务执行”是同一个工程习惯。区别只在于，Python 给了你官方标准能力，而不是让每个框架自己重复造一套。

第三层是“重新分发调用”。

PEP 362 里的 `BoundArguments` 很值得你建立直觉。它不是单纯“校验成功”的标记，而是把最终参数映射结果保存成 `arguments`，并且还能导出成 `args` / `kwargs` 再次调用函数。这个设计对 Agent tool runner 很实用：

1. 宿主先绑定参数。
2. 中间层可以补默认值、做类型转换、做 trace 注入。
3. 最后再用整理后的 `args` / `kwargs` 真正执行工具。

这比从头到尾把一堆 `dict` 到处传要稳很多，因为“Python 自己怎么看这次调用”已经被固定下来了。

第四层是“装饰器、包装器和工具框架为什么还能保住签名”。

官方文档和 PEP 都提到 `__wrapped__`、`__signature__`。这里可以把它理解成两类能力：

1. `__wrapped__`：告诉 introspection 系统“继续往里看，真正的原函数在里面”。
2. `__signature__`：告诉 introspection 系统“不要猜了，直接用我显式给出的签名”。

这和 Agent 工程关系很近。因为你做工具系统时，往往不会直接把原函数裸奔出去，而是会包一层：

1. 做鉴权。
2. 做日志和 trace。
3. 做输入输出 schema 转换。
4. 做重试、超时、并发控制。

如果这层包装把签名弄丢了，后面的自动 schema 生成、参数校验、文档生成就会一起变差。所以很多成熟框架都很在意保留 `__wrapped__`，必要时还会显式设置 `__signature__`。

第五层是“不要误以为内省总是可靠且免费”。

这组资料很克制地提醒了两个现实边界：

1. 某些内建/C 扩展函数本来就不一定能提供完整签名。
2. `Signature` 不会自动缓存，框架要自己决定什么时候缓存、缓存多久、缓存失效条件是什么。

这点很像你以前做客户现场集成时的经验：反射很好用，但不能把它当成零成本、零歧义、零边界的万能方案。到了 Python Agent 工程里，姿势还是一样的：先吃透官方契约，再决定哪些地方适合自动化，哪些地方需要你显式覆盖。

## 3 key takeaways

1. `inspect.signature()` 不是语法糖，它是 Python 里读取工具契约、做参数校验、生成 schema 的底层公共接口。
2. `Signature.bind()` / `BoundArguments` 的价值，不只是“看看能不能调”，而是把一次调用先正规化，再交给后续执行链处理。
3. `__wrapped__` 和 `__signature__` 决定了包装后的函数还能不能保住原始契约；这直接影响 Agent 工具注册、文档生成和运行时校验质量。

## relation to Agent engineering

这篇内容和 Agent engineering 的关系非常直接，而且正好能补你当前的 Python 缺口。

第一，工具注册层。很多 Agent 框架允许你“给它一个 Python 函数，它就自动变成 tool”。这背后第一步通常就是 `inspect.signature(func)`，然后把参数列表转成内部契约对象，再映射到 JSON Schema 或 provider-specific tool schema。

第二，tool runner 层。模型给出的参数往往先是一个 JSON 对象，不会直接拿来裸调。成熟宿主会先做：

1. schema 校验。
2. `bind()` 级别的 Python 调用规则校验。
3. 默认值补全。
4. 必要的类型转换或上下文注入。

这跟你以前做企业接口适配时的“请求报文校验 -> 参数装配 -> 服务调用”流程非常像。你要迁移的不是能力本身，而是实现语言和生态。

第三，装饰器与可观测性层。真实 Agent 工具很少是“纯函数直出”，而是经常被 trace、权限、熔断、缓存、重试层包起来。你理解 `follow_wrapped`、`__wrapped__`、`__signature__` 之后，再去看框架为什么能在包装之后仍然展示正确签名，就会顺很多。

第四，MCP / tool contract 桥接层。MCP、Structured Outputs、OpenAI tool calling 这些机制最后都要落回宿主里的真实函数执行。`Signature` 这层就是 Python 世界里“从外部契约落到本地调用规则”的桥。

## a small action for tonight

今晚做一个 30 分钟的小实验，目标不是写大工程，而是把“读取签名 -> 绑定参数 -> 执行函数”这条链亲手走通：

1. 写 3 个函数：一个带默认值，一个带 keyword-only 参数，一个带 `*args/**kwargs`。
2. 对每个函数打印 `inspect.signature()`、`sig.parameters`、每个 `Parameter.kind/default/annotation`。
3. 对每个函数分别写一组合法调用和一组非法调用，用 `sig.bind()` 观察 `TypeError` 是怎么报的。
4. 试一次 `ba = sig.bind(...)` 之后 `ba.apply_defaults()`，看看默认值是怎么被补进 `ba.arguments` 的。
5. 再写一个带装饰器的函数，分别观察默认 `follow_wrapped=True` 和 `False` 的差别。

如果你今晚只做完这一个实验，后面再看任意一个“自动把函数注册成 Agent tool”的 Python 框架，源码理解速度会快很多。

## 原文关键段落翻译（人工翻译，放在文末）

1. `Signature` 对象表示一个可调用对象的调用签名以及返回注解；要获取它，应使用 `signature()` 函数。
2. `signature()` 可以处理很广的一类 Python 可调用对象，从普通函数、类，到 `functools.partial()` 对象都可以。
3. 函数签名中的 `/` 表示它前面的参数是 positional-only，只能按位置传参。
4. 某些可调用对象在某些 Python 实现里可能无法被完整内省；例如 CPython 中部分由 C 实现的内建函数并不提供足够的参数元数据。
5. `Signature.bind(*args, **kwargs)` 会把实参与形参建立映射；如果参数不匹配，就抛出 `TypeError`。
6. `BoundArguments` 保存绑定后的参数映射；如果需要把依赖默认值的参数也补进去，可以调用 `apply_defaults()`。
7. `signature()` 的实现顺序会优先考虑 `__signature__`，然后再看 `__wrapped__`、普通函数、绑定方法、`partial`、类及其 `__call__` / `__new__` / `__init__`。
8. `Signature` 是惰性创建的，不会自动缓存；如果调用方需要缓存，应显式地自己做。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.python.org/3.13/library/inspect.html
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://peps.python.org/pep-0362/
