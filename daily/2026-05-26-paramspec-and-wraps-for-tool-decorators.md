# title

2026-05-26 为什么 `ParamSpec + @wraps` 是 Python Agent 工具装饰器不丢签名的关键

## original source

- 标题：`typing — Support for type hints`（`ParamSpec` / `Concatenate`）
- 链接：https://docs.python.org/3.13/library/typing.html
- 来源类型：Python 官方文档
- 访问时间：2026-05-26

- 标题：`functools — Higher-order functions and operations on callable objects`（`update_wrapper` / `wraps`）
- 链接：https://docs.python.org/3.12/library/functools.html
- 来源类型：Python 官方文档
- 访问时间：2026-05-26

- 标题：`PEP 612 – Parameter Specification Variables`
- 链接：https://peps.python.org/pep-0612/
- 来源类型：Python 官方 PEP / 类型系统设计说明
- 访问时间：2026-05-26

## why read today

你这几天已经连续补了 `inspect.signature()`、`Annotated`、`TypedDict`、JSON Schema、`unittest.mock`。这些内容分别解决了：

1. 函数签名长什么样
2. 参数元数据怎么挂
3. schema 怎么描述
4. 工具怎么测

但真实 Agent 工程里还有一个特别容易把前面成果全部“包坏”的地方：装饰器。

一旦你开始给 tool 函数加重试、打点、鉴权、超时、日志、trace、会话注入，代码很快就会变成：

`@trace_tool`
`@retry`
`@with_request_context`
`async def search_docs(...) -> ...`

如果你只会写 `*args, **kwargs` 风格的包装器，不理解 `ParamSpec`、`Concatenate` 和 `@wraps`，就会出现几个很典型的问题：

1. 类型检查器看不出包装前后的参数关系
2. 工具 schema 生成器读到的是一个模糊的 wrapper
3. trace / eval / registry 侧内省拿不到原函数信息
4. 你以为自己只是“加了个中间层”，实际上把工具契约弄松了

这对你尤其重要，因为你不是缺“服务端工程感”，而是缺 Python 这套“高阶函数 + 反射 + 类型提示”如何一起工作。Java/Spring 里这件事常由 AOP、注解、代理类完成；Python 里更常见的落点就是装饰器、`__wrapped__`、`inspect` 和 `typing`。

## original-text translation

Python 官方 `typing` 文档给 `ParamSpec` 和 `Concatenate` 的定位非常明确：它们是给“接收其他可调用对象作为参数的高阶函数”用的，尤其适合描述“包装后仍然依赖原函数参数列表”的场景。普通的 `Callable[..., Any]` 虽然能写出来，但它把参数类型细节全部抹平了，导致包装器内部的 `*args`、`**kwargs` 只能退化成 `Any`，类型检查器也就失去了约束力。

文档进一步说明，`Concatenate` 可以描述“装饰器给原函数额外加了一个参数”或“包装器隐藏了一个参数”的情况；而 `P.args`、`P.kwargs` 则让你在 wrapper 里继续把这组参数原样传下去。也就是说，`ParamSpec` 不是为了好看，而是为了保住“这个包装器和原函数签名之间到底是什么关系”。

`functools` 文档则解释了另一半问题：即便你在类型层面保住了参数关系，如果不做 `update_wrapper()` / `@wraps`，返回出来的函数元数据仍然会像 wrapper 自己，而不像原函数。官方明确说，它的主要用途就是装饰器场景；否则，返回函数的 `__name__`、`__doc__`、`__annotations__` 等信息会反映 wrapper 的定义，而不是原函数定义。

更关键的是，官方文档强调 `update_wrapper()` 会自动加上 `__wrapped__`。这个属性不是可有可无的小细节，而是内省链条里的关键挂钩：很多调试、反射、再包装场景都靠它回到原函数。

PEP 612 把设计动机讲得更直接：在没有参数规格变量之前，装饰器很难同时做到“足够通用”与“保留被装饰函数的参数约束”。`ParamSpec` 要解决的，不是语法便利，而是让这种依赖关系能够被静态表达出来。

## Chinese deep summary

如果把今天这篇材料压成一句话，那就是：`ParamSpec` 解决“包装器类型上如何不丢原函数参数契约”，`@wraps` 解决“包装器运行时如何不丢原函数身份信息”，两者合起来，才够支撑一个像样的 Python Agent tool 装饰器体系。

第一层，要把 `ParamSpec` 看成 Python 版“保留方法签名关系”的类型工具，而不是高级写法。

很多初学 Python 的工程师写装饰器时，默认是：

```python
def log_calls(fn):
    def wrapper(*args, **kwargs):
        ...
        return fn(*args, **kwargs)
    return wrapper
```

运行当然能跑，但从工程角度看，它已经悄悄把契约弄丢了一半。类型层面上，wrapper 的参数只是“某些 args/kwargs”；如果你后面还要基于这个函数自动生成 tool schema、校验调用、做 IDE 提示、给团队看代码，这种写法就太松了。

`ParamSpec` 的价值是：你可以明确表达“wrapper 接收的参数，和被包装函数接收的是同一组参数”。这和你过去在 Java 里强调接口方法签名一致，本质上是同一种契约意识，只是 Python 通过类型变量表达。

第二层，`Concatenate` 让你能把 Agent 工程里常见的“宿主注入参数”写清楚。

这点对 Agent 非常关键，因为真实工具包装器经常不是简单透传，而是会：

1. 在最前面塞一个 context / session / lock / tracer
2. 对外隐藏这个宿主参数，不让模型来填
3. 仍然保留业务参数列表不变

这正是 `Concatenate[HostContext, P]` 擅长表达的东西。它非常像你做系统集成时区分“平台侧注入字段”和“客户真正传入字段”的思路。只不过这里不是 XML/JSON 契约，而是 Python callable 契约。

第三层，`@wraps` 不是礼貌性装饰，而是反射链条的基础设施。

很多人知道不用 `@wraps` 时函数名会变成 `wrapper`，但低估了这个问题的工程后果。对 Agent 系统来说，函数元数据经常会被多处消费：

1. tool registry 读名字和文档
2. schema builder 读注解
3. trace 系统读函数身份
4. 调试工具顺着 `__wrapped__` 追原函数

如果这里丢掉了，后果不是“文档不优雅”，而是整条工具内省链会变得模糊，严重时还会影响 schema、日志和可观测性。

第四层，只保 `@wraps` 不保 `ParamSpec`，或者只保 `ParamSpec` 不保 `@wraps`，都不够。

这是今天最值得建立的组合心智。

只用 `@wraps`：
运行时元数据保住了，但类型系统依然不知道 wrapper 和原函数参数之间的关系。对自动补全、静态检查、复杂装饰器堆叠都不够稳。

只用 `ParamSpec`：
类型写得很漂亮，但运行时如果不拷贝元数据、不挂 `__wrapped__`，很多内省工具依然只能看到 wrapper 壳子。

Agent 工程偏偏两边都要，因为它既依赖“静态契约清晰”，又依赖“运行时反射可追踪”。

第五层，这正是你把 Java/Spring 经验迁到 Python Agent 工程时最该补的一环。

你并不缺对“横切关注点”的理解。重试、日志、鉴权、上下文注入、调用追踪，这些你过去都做过。你现在缺的是：在 Python 里，这些横切逻辑最容易通过装饰器落地，而装饰器一旦写糙，就会把类型和内省链一起搞坏。

所以今天这篇材料，本质上是在补一条迁移映射：

1. Java 注解 / AOP / 代理
2. 对应到 Python 装饰器 / `ParamSpec` / `__wrapped__`
3. 再落到 Agent 的 tool 注册、schema 生成、trace 包装、上下文注入

这条映射一旦建立，你看很多 Python Agent 框架源码时就不会再觉得“为什么到处都在套 decorator”，而会知道它们是在维护工具契约。

## 3 key takeaways

1. `ParamSpec` 的核心价值不是“让装饰器写法更高级”，而是把 wrapper 与原函数之间的参数依赖关系保留下来，避免 `Callable[..., Any]` 把契约抹平。
2. `Concatenate` 很适合表达 Agent 工具里“宿主注入一个隐藏参数、对外仍保留业务参数签名”的模式，这正对应 context、session、tracer 一类包装场景。
3. `@wraps` / `update_wrapper()` 的真正工程价值在于保住 `__name__`、`__annotations__`、`__doc__` 和 `__wrapped__`，让 tool registry、schema builder、trace 与调试工具还能找到原函数。

## relation to Agent engineering

这篇内容和 Agent engineering 的关系非常直接，而且正落在你当前最需要补的 `Python 工程化 + Tool Calling + MCP` 交叉点上。

第一，tool 装饰器设计。只要你给工具加日志、鉴权、重试、超时、限流、trace，就已经进入装饰器地带。没有 `ParamSpec + @wraps`，这些包装层很容易把工具契约越包越糊。

第二，tool schema 生成。很多 Agent 宿主会从 Python 函数签名和注解自动生成 schema。只要 wrapper 没处理好，生成器看到的就不一定还是原始业务函数。

第三，MCP / Host 注入。MCP host 或自定义 agent runtime 很常见的一种写法，就是运行时给工具塞入 session、request context、资源句柄或 tracer，而模型侧并不知道这个参数存在。`Concatenate` 正好能把这件事在类型层面讲清楚。

第四，eval 与 trace。要做可重复评测和故障排查，前提是你知道“当前执行的到底是哪一个工具函数”。`__wrapped__` 这类元数据挂钩，表面像小事，实际上决定了你能不能稳定地把执行壳子还原回业务函数。

第五，帮助你迁移 Java 经验。你过去在企业集成里很熟悉“横切逻辑不能破坏主契约”。今天这篇材料只是把这个原则翻译成 Python 的表达方式。

## a small action for tonight

今晚做一个 35 分钟的小练习，目标不是写大项目，而是亲手感受到“保签名”和“不保签名”的差别：

1. 先写一个原始工具函数：`async def search_docs(query: str, top_k: int = 5) -> str: ...`
2. 写第一个装饰器版本：只用 `*args, **kwargs` 包装，不加 `@wraps`，然后打印它的 `__name__`、`__doc__`、`inspect.signature()`。
3. 写第二个版本：加上 `@wraps`，观察运行时内省结果怎么恢复。
4. 写第三个版本：用 `ParamSpec` 和返回类型变量把装饰器标出来，再让编辑器或类型检查器看 wrapper 调用时还能不能保留参数提示。
5. 如果还有 10 分钟，再尝试写一个“宿主自动注入 tracer，但调用方不用传 tracer”的装饰器，练一次 `Concatenate`。

如果你今晚只把这一个练习做完，后面再看 Python Agent 框架里的 `@tool`、`@trace`、`@with_context`、`@retry` 这类包装器，理解会明显快很多。

## 原文关键段落翻译（人工翻译，放在文末）

1. `ParamSpec` 用来描述这样的高阶可调用对象：它们接收另一个可调用对象，而参数类型之间存在依赖关系。
2. `Concatenate` 可以和 `Callable`、`ParamSpec` 一起使用，用来标注那种会给另一个可调用对象增加、移除或变换参数的高阶可调用对象。
3. 没有 `ParamSpec` 时，常见替代写法是 `Callable[..., Any]`，但这样一来类型检查器无法正确检查 wrapper 里的 `*args` 和 `**kwargs`，很多地方会退化成 `Any`。
4. `update_wrapper()` 的主要用途，是给装饰器返回的 wrapper 补齐原函数的外观；否则，返回函数的元数据反映的是 wrapper 自己，而不是原函数。
5. `update_wrapper()` 会自动给 wrapper 增加 `__wrapped__`，这样做是为了让内省和其他场景还能访问到原始函数。
6. `@wraps` 只是 `update_wrapper()` 的便捷写法，适合在定义 wrapper 时直接使用。
7. PEP 612 的核心动机之一，是让装饰器既能保持通用性，又能保留被装饰函数的参数约束，而不是在类型层面把这层关系丢掉。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.python.org/3.13/library/typing.html
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.python.org/3.12/library/functools.html
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://peps.python.org/pep-0612/
