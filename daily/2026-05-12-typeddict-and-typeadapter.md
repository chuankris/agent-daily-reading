# title

2026-05-12 为什么 `TypedDict` + `TypeAdapter`，是你把 Agent 输入输出契约写轻、写稳、写到位的关键一课

## original source

- 标题：`typing` — Support for type hints — Python 3.14.5 documentation
- 链接：https://docs.python.org/3/library/typing.html
- 来源类型：官方文档
- 访问时间：2026-05-12

- 标题：Type Adapter | Pydantic Docs
- 链接：https://pydantic.dev/docs/validation/latest/concepts/type_adapter/
- 来源类型：官方文档
- 访问时间：2026-05-12

## why read today

这几天你已经连续补了 Pydantic 基础模型、`contextvars`、MCP、Agent handoff 和 run context。下一步如果想把这些东西真正写成“像样的工程”，一个非常现实的问题会跳出来：

不是所有输入输出，都值得先上一个完整 `BaseModel`。

你做了 10 年 Java / Spring / IoT 集成，应该很熟悉这种场景：有些对象是正式 DTO，要有完整类定义；但也有很多东西其实只是协议边界上的“轻载体”，比如一段工具入参、一个中间层 payload、一个资源描述、一个运行快照里的局部结构。如果在 Python 里这些东西全部退化成裸 `dict`，系统会很快变脏；但如果每个小结构都建成 `BaseModel`，代码又会开始变重。

今天这两个官方源正好补上这个中间地带：

1. `TypedDict` 解决“字典长什么样”这件事，让 key 集合、必填/选填、嵌套结构变成显式契约。
2. `TypeAdapter` 解决“怎么把这种轻量类型真正拿来做校验、序列化、JSON Schema 输出”这件事。

对 Agent 工程来说，这一层非常重要。因为很多边界天生就不是数据库实体，也不是完整领域对象，而是：

1. tool 参数
2. tool 返回片段
3. eval case 的局部字段
4. MCP 消息里的某个 payload
5. session state 中的一小段可恢复结构

如果你能把这些边界用 `TypedDict` + `TypeAdapter` 稳住，后面再接 Structured Outputs、tool schema、FastAPI、MCP adapter 时，代码会明显更轻、更准，也更接近“交付型系统”而不是“能跑就行的 demo”。

## original-text translation

Python 官方 `typing` 文档先把 `TypedDict` 的边界讲得很清楚：它本质上是“给字典加类型提示”的特殊结构。运行时里它仍然只是普通 `dict`，不会自动做检查；真正强约束它的，是类型检查器。文档强调两点。第一，`TypedDict` 不是“随便一个 dict”，而是一个预期拥有固定 key 集合、且每个 key 对应固定值类型的字典契约。第二，默认所有 key 都必须存在，但你可以用 `NotRequired` 把单个字段标成可缺省，也可以用 `total=False` 让整个字典默认变成“可选键集合”。

这件事很像你熟悉的接口契约分层。不是所有结构都值得升格为完整对象，但只要一个字典已经跨越模块、跨越进程边界，或者要进入 LLM / tool / API / queue / snapshot 这样的工程边界，它就不该再是“随手拼的 map”，而应该至少是一个显式声明过字段要求的 `TypedDict`。

Pydantic 的 `TypeAdapter` 文档则补上了关键下一步：光有类型声明还不够，你还需要把这些轻量类型真正拿来做校验、序列化和 schema 生成。官方文档说得很直接：如果你想验证的对象不是 `BaseModel`，或者你想直接验证 `list[SomeModel]` 这样的组合类型，`TypeAdapter` 就是那个合适的入口。它不要求你为了校验一个简单结构专门造一个完整模型类。

文档给了一个非常贴近 Agent 工程的例子：定义一个 `TypedDict` 版的 `User`，然后用 `TypeAdapter(list[User])` 去校验一组字典。字符串形式的 `"3"` 可以被解析成整数 `3`，非法值会得到明确的校验错误，最后还能直接输出 JSON。也就是说，`TypedDict` 负责描述“这类字典应当长什么样”，`TypeAdapter` 负责把这个描述变成真正可运行的边界设施。

文档还提醒了两个工程细节。第一，`TypeAdapter.dump_json()` 返回的是 `bytes`，这和 `BaseModel.model_dump_json()` 返回 `str` 不一样，意味着你在网络传输、缓存、消息队列场景里可以少做一层无意义的转换。第二，创建 `TypeAdapter` 时会有 schema 构建成本，所以同一种类型的 adapter 最好创建一次后复用，而不是在循环里反复 new。

如果把两篇文档合起来看，结论其实非常实用：`BaseModel` 适合正式的数据对象；`TypedDict` + `TypeAdapter` 适合那些“还没有重到需要建模型类，但已经重到不能继续裸奔”的协议边界。

## Chinese deep summary

今天这组材料最值得你吸收的，不是某个语法点，而是一种 Python Agent 工程里的边界分层思路：

1. 不是所有结构都要上 `BaseModel`
2. 但只要跨工程边界，就不该停留在裸 `dict`
3. `TypedDict` + `TypeAdapter` 正好填上这块中间层

这和你过去做 Java 集成项目时的经验是高度一致的。真正成熟的交付系统，不会把所有东西都做成重量级领域对象，但也不会让关键边界长期停留在 `Map<String, Object>`。大家通常会按场景分层：

1. 核心领域对象，用正式类
2. 接口协议对象，用轻量 DTO
3. 临时载荷，也至少要有清晰字段契约

Python 世界里，很多人一开始因为 `dict` 太顺手，会把第二层和第三层直接省掉。Agent 工程尤其容易这样，因为模型返回 JSON、tool 入参是 JSON、MCP payload 也是 JSON、评测样本还是 JSON，于是所有东西都滑进“先收个 dict 再说”的黑洞。短期很快，长期非常痛苦。

为什么痛苦？因为一旦进入下面这些场景，裸 `dict` 会迅速失控：

1. 不同 tool 共享一套参数片段，但字段要求略有不同
2. 人工补录或恢复运行时，缺字段和脏字段混进来
3. 同一个 payload 在多个模块之间透传，没人说得清哪些 key 是必须的
4. 你想给 LLM 输出、tool 入参、FastAPI 请求体、eval 样本复用同一份结构定义

`TypedDict` 的价值，就是让“字典契约”从口头约定变成代码里的显式结构。它仍然保留了字典的轻量和灵活，不会像完整模型那样引入太多样板，但已经足够表达：

1. 哪些字段必须有
2. 哪些字段可以缺省
3. 每个字段应该是什么类型
4. 这个结构是不是某个更大结构的一部分

这一步对你这种 Java 背景的人尤其关键，因为你很容易在 Python 里走向两个极端：

1. 要么把一切都重建成完整类，结果开发成本偏高
2. 要么看到 `dict` 很方便，就把契约意识丢掉

`TypedDict` 给出的，是第三条路：保留轻量，但不放弃结构。

但只声明结构还不够。真正把它变成工程设施的是 `TypeAdapter`。

这里你可以把 `TypeAdapter` 理解成“给任意 Python 类型补上 Pydantic 校验和序列化能力的适配器”。它最有价值的地方在于：你不必为了一个轻量载荷专门造 `BaseModel`，也不用退回手写 `if` / `try` / `isinstance`。你可以直接说：

1. 这个输入应该是 `TypedDict`
2. 这个返回应该是 `list[TypedDict]`
3. 这个结构应该能导出 JSON Schema
4. 这个 payload 应该严格经过一次校验再进入执行层

对 Agent 工程来说，这种能力几乎正中红心。因为很多最关键的对象，其实都不是 ORM 实体，也不是完整业务类，而是“运行过程中的协议对象”。比如：

1. `search_tool` 的入参：`query`、`top_k`、`filters`
2. `approval_request` 的载荷：`reason`、`risk_level`、`requested_by`
3. `eval_case` 的局部结构：`input`、`expected_tool`、`metadata`
4. `mcp_resource_ref` 的片段：`uri`、`mimeType`、`title`

这些对象有三个共同点：

1. 很重要
2. 很轻量
3. 很容易被人偷懒写成 `dict`

而 `TypedDict` + `TypeAdapter` 正是在这里最值钱。

再往深一点说，这组材料帮你建立的是一套“轻重有别”的 schema 观，而不是“只有一种正确建模方式”。

你可以这样理解：

1. `BaseModel`：适合正式输入输出对象、配置对象、状态对象
2. `TypedDict`：适合轻量协议载荷、模块间字典契约、嵌套片段
3. `TypeAdapter`：把上述任意类型真正接上校验、序列化、Schema 生成

一旦这套分层立住，你后面写 Agent host 会顺很多。你会自然知道：

1. 哪些对象需要完整模型
2. 哪些对象只需要轻量结构契约
3. 哪些地方该在执行前做统一校验
4. 哪些 schema 应该被复用给 tool、API、eval 和恢复流程

这其实就是成熟后端工程思维迁移到 Python Agent 世界里的一个关键标志：不是“会不会写 Pydantic”，而是“知道该在什么边界上用多重的契约”。

## 3 key takeaways

1. `TypedDict` 不是运行时校验工具，而是轻量字典契约工具：它让必填键、可选键和字段类型先被明确定义出来，避免关键 payload 长期裸奔。
2. `TypeAdapter` 把任意 Python 类型接上 Pydantic 的校验、序列化和 JSON Schema 能力，因此特别适合 `TypedDict`、`list[...]`、嵌套组合类型这类“没必要单独建 `BaseModel`”的边界对象。
3. 对 Agent 工程来说，`BaseModel` 不是唯一答案；更实用的做法是按边界重量分层，用 `BaseModel` 管正式对象，用 `TypedDict` + `TypeAdapter` 管轻量协议对象。

## relation to Agent engineering

这组内容和 Agent 工程的关系非常直接，而且比普通 Web 后端更明显。

第一层关系，是 tool contract。模型生成出来的 tool 参数，本质上就是一段要进入执行层的协议载荷。很多团队会直接收 JSON 再手动解；但更稳的做法，是先用 `TypedDict` 描述参数结构，再用 `TypeAdapter` 做统一校验。这样你就把“LLM 说了什么”和“系统真正允许执行什么”分开了。

第二层关系，是 schema 复用。Agent 系统里，同一份结构常常会同时出现于 prompt 约束、tool schema、FastAPI 接口、MCP payload、eval case、恢复快照。如果这些地方各写各的，很快就会漂移。用轻量类型加统一 adapter，能明显降低漂移成本。

第三层关系，是 Java 到 Python 的迁移质量。你过去熟悉的不是某个框架，而是“关键边界不能糊”。今天这组内容，等于是在 Python 里给你补上一个相当于“轻量 DTO + 统一校验入口”的答案。它没有 Java 那么重，但工程含义是同一类东西。

第四层关系，是保持系统轻量。很多 Agent 项目一旦开始“工程化”，很容易过度建模，结果代码读起来比业务本身还重。`TypedDict` + `TypeAdapter` 给了你一个更均衡的落点：比裸 `dict` 严格，比完整类更轻，正适合当前这个从 demo 往可维护系统过渡的阶段。

## a small action for tonight

今晚做一个 30 分钟小实验，目标不是写功能，而是把“轻量契约”真正跑通：

1. 定义一个 `TypedDict`：`SearchArgs`，字段包含 `query: str`、`top_k: int`、`filters: NotRequired[list[str]]`。
2. 用 `TypeAdapter(SearchArgs)` 校验一份字典，分别试三种输入：正确输入、`top_k=\"3\"`、漏掉 `query`。
3. 再定义 `TypeAdapter(list[SearchArgs])`，模拟一次批量 tool 调用参数校验。
4. 打印 `json_schema()` 看看它长什么样，再对照你之前读过的 Structured Outputs / tool schema 内容，想一想哪些边界可以直接共用。
5. 最后问自己一句：在你未来的 Agent host 里，哪些对象应该继续是 `BaseModel`，哪些对象其实更适合先落成 `TypedDict`？

## 原文关键段落翻译（人工翻译，放在文末）

1. `TypedDict` 是一种给字典补充类型提示的特殊结构；在运行时里，所谓 `TypedDict` 实例其实仍然只是普通字典。
2. `TypedDict` 声明的是一种字典类型：它预期实例拥有一组固定 key，并且每个 key 对应一致的值类型；这类约束默认由类型检查器而不是 Python 运行时执行。
3. 默认情况下，`TypedDict` 的所有 key 都必须存在；如果想让某些 key 可以缺省，可以使用 `NotRequired`。
4. Pydantic 提供 `TypeAdapter`，用于在不创建 `BaseModel` 的前提下，对任意 Python 类型做校验、序列化和 JSON Schema 生成。
5. `TypeAdapter` 特别适合验证那些不是 `BaseModel` 的类型，或者像 `list[SomeModel]` 这样的组合类型。
6. 在官方示例里，`TypeAdapter(list[User])` 可以把 `TypedDict` 列表做解析和校验，像字符串形式的数字也可以被转换成整数。
7. `TypeAdapter.dump_json()` 返回的是 `bytes`；这和 `BaseModel.model_dump_json()` 返回 `str` 不同。
8. 创建 `TypeAdapter` 时需要把目标类型分析并转换为内部 schema，这个过程有实际成本，所以同一种 adapter 更适合复用，而不是在热点路径里反复创建。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.python.org/3/library/typing.html
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://pydantic.dev/docs/validation/latest/concepts/type_adapter/
