# title

2026-05-13 为什么你该把 `JSON Schema` 当成 Agent 工程里的“统一契约层”，而不是只把它当成一段给模型看的 JSON

## original source

- 标题：JSON Schema Reference: `object`
- 链接：https://json-schema.org/understanding-json-schema/reference/object
- 来源类型：官方文档
- 访问时间：2026-05-13

- 标题：JSON Schema | Pydantic Docs
- 链接：https://pydantic.dev/docs/validation/dev/concepts/json_schema/
- 来源类型：官方文档
- 访问时间：2026-05-13

## why read today

你昨天刚补完 `TypedDict` + `TypeAdapter`，已经知道“轻量字典契约”在 Python Agent 工程里很重要。但如果只停在 Python 类型层，你还差最后一块关键拼图：

怎么把这份契约稳定地复用到模型输出约束、tool 参数、FastAPI 接口、MCP payload、eval 样本和日志回放里？

答案通常不是再多建几个类，而是理解 `JSON Schema` 这一层。

对你这种做了 10 年 Java / Spring / IoT 集成 / ToB 交付的人来说，这篇最值得读的点在于：`JSON Schema` 不是“前端表单校验小工具”，而是跨边界数据契约。它解决的是“不同模块、不同进程、不同系统，如何对同一份 JSON 结构说同一种话”。

Agent 工程里，这件事尤其重要。因为你后面真正要长期维护的，不只是 prompt，而是下面这些边界：

1. tool calling 的参数结构
2. structured output 的结果结构
3. MCP tool / resource / prompt 的载荷结构
4. eval case 的输入输出样本
5. session state、checkpoint、回放记录里的 JSON 片段

如果这些边界只有 Python 类型定义而没有一个更通用的 schema 层，系统会很快出现两类问题：

1. 代码里“以为”结构是 A，模型和接口层“实际”执行的是 B
2. 同一份契约在不同地方各写一版，几周后开始漂移

今天这两份官方材料刚好把这个断点补上：

1. JSON Schema 官方文档告诉你，`properties`、`required`、`additionalProperties`、`unevaluatedProperties` 这些关键词到底意味着什么。
2. Pydantic 官方文档告诉你，Python 里的 `BaseModel` 和 `TypeAdapter` 怎样导出 JSON Schema，以及“校验模式”和“序列化模式”为什么不能混着理解。

这正好是你从“会写 Python 类型”走向“会设计 Agent 契约层”的一步。

## original-text translation

JSON Schema 官方对象章节先讲了一个很容易被忽略、但对工程边界特别重要的事实：在 JSON 里，对象就是字符串 key 到 value 的映射；它和 Python `dict` 很像，但 key 必须是字符串。接着文档把对象契约拆成几层。`properties` 用来声明“已知字段各自该满足什么 schema”，但它本身并不代表这些字段一定要出现；如果要把某些字段设成必填，必须显式写 `required`。这意味着“字段被定义”与“字段必须存在”是两件不同的事。

文档还强调了另一个很多工程实现会踩坑的默认值：即使你已经定义了 `properties`，额外字段默认仍然是允许的。也就是说，只写了字段列表，不等于这个对象是“封闭对象”。如果你不想让模型、上游服务或者调用方偷偷塞进未声明字段，就要进一步用 `additionalProperties: false`，或者给额外字段再定义一层约束。

然后文档继续说明了一个更深的点：`additionalProperties` 并不是“全局关闭未知字段”的万能开关。它只识别和它处在同一个子 schema 里的字段声明，所以一旦你用 `allOf` 等组合方式扩展 schema，`additionalProperties` 很容易把后来拼上的字段也当成“额外字段”。这就是为什么较新的 `unevaluatedProperties` 很重要，它能识别已经在其他子 schema 中被成功验证过的字段，更适合做可组合、可扩展的封闭对象约束。

Pydantic 的 JSON Schema 文档则把 Python 工程侧的桥补上。官方文档明确说，`BaseModel.model_json_schema()` 和 `TypeAdapter.json_schema()` 返回的是“可 JSON 化的 schema 字典”，它们描述的是结构契约；而 `model_dump_json()`、`dump_json()` 返回的则是实例序列化后的 JSON 字符串。这两类 API 名字很像，但责任完全不同，一个管“对象应该长什么样”，一个管“当前对象长成了什么字符串”。

文档还补了一个对 Agent 工程很关键的细节：生成 JSON Schema 时可以指定 `mode`。默认是 `validation`，表示 schema 对应“输入校验”视角；你也可以切到 `serialization`，表示 schema 对应“输出序列化”视角。两者在某些类型上会不同，例如某个字段在输入阶段可以接受更宽松的形式，但序列化出去时结构会更收敛。这提醒你：不要把“系统愿意接收什么”与“系统最终承诺输出什么”混成一回事。

最后，Pydantic 文档还说明，生成出来的 schema 默认遵循 JSON Schema Draft 2020-12，并且会把复用的子模型放进 `$defs`，通过 `$ref` 做引用。这背后的工程含义是：一旦你把 Python 边界对象认真建模，就有机会让同一份结构定义同时服务于 tool schema、接口文档、评测样本校验和跨模块契约，而不是每层各写一遍。

## Chinese deep summary

今天这组内容最值得你真正吸收的，不是几个 schema 关键词，而是一种“统一契约层”思维。

很多刚做 Agent 的工程会把注意力放在 prompt 和模型能力上，但系统一旦进入可维护阶段，最先暴露问题的往往不是提示词，而是契约漂移：

1. tool 定义里说字段是可选的，执行层代码却默认它必填
2. API 接口接受额外字段，状态恢复逻辑却在遇到这些字段时崩掉
3. eval 样本里沿用旧结构，生产代码里的 schema 已经换了
4. prompt 里写的是一种 JSON 形状，Pydantic 模型里又是另一种

这些问题你在过去做集成项目时其实见得很多，只是当时它们可能表现为 DTO、报文协议、设备上送 payload、B 端接口字段对齐。换到 Agent 工程，本质没变，仍然是那句话：

跨边界的结构，不能靠默契。

而 `JSON Schema` 的价值，就在于它把“默契”变成可共享的正式契约。

为什么这件事在你现在的阶段特别关键？因为你最近补的内容已经形成了一条很清晰的链：

1. `pyproject`、`pytest`、`asyncio`、`FastAPI` 是承载层
2. Pydantic、`TypedDict`、`TypeAdapter` 是 Python 侧的边界建模层
3. tool calling、MCP、eval、trace 会不断消耗这些边界定义

如果没有 `JSON Schema` 这层，你后面很容易出现一个典型问题：Python 代码里结构看起来很清楚，但一旦结构要走出 Python 运行时，契约就开始散掉。

举个很贴近 Agent host 的例子。假设你定义了一个工具参数：

1. `query: str`
2. `top_k: int`
3. `filters: list[str] | None`

在 Python 里这可以是 `BaseModel`，也可以是 `TypedDict` + `TypeAdapter`。但真正系统化之后，你还会立刻问：

1. LLM 输出时，哪些字段必须给？
2. 是否允许额外字段，比如 `debug=true`？
3. `filters` 缺省和 `filters=null` 是不是一回事？
4. 这个结构能不能直接复用到 API、MCP、eval 和回放校验？

这些问题已经不是“类型标注”能单独回答的，而是 schema 设计问题。也正因为如此，`properties`、`required`、`additionalProperties` 这些看起来很基础的东西，实际上是 Agent 工程稳定性的地基。

这里最值得你提高警惕的是两个默认行为。

第一，定义了 `properties` 不代表字段必填。

这和很多人直觉不一样。很多工程师看到 schema 里列出了 `query` 和 `top_k`，就以为这两个字段天然必填，但 JSON Schema 不是这么工作的。你必须再写 `required`。这对 tool calling 很重要，因为模型输出少字段时，你得知道这是“结构允许的缺省”，还是“真正的无效调用”。

第二，定义了 `properties` 也不代表额外字段会被拒绝。

默认情况下，多出来的字段是允许的。这在普通 Web 表单里可能问题不大，但在 Agent 系统里往往会埋雷。因为额外字段经常意味着：

1. 模型开始幻觉式地补参数
2. 上游版本变了，但下游没意识到
3. 恢复旧状态时混入了新旧格式
4. 某个中间环节偷偷塞了调试字段

如果你的目标是做“交付型系统”，很多关键对象都应该显式决定：到底要不要关闭额外字段。不是一律关死，但必须是有意识的设计，而不是吃默认值。

再进一步，官方文档对 `additionalProperties` 和 `unevaluatedProperties` 的区分，其实很适合你这种做过复杂集成的人理解。它本质上是在提醒你：

一个 schema 不只是“静态字段表”，它还可能被组合、继承、扩展。

你过去在 Java 集成项目里可能做过主协议 + 扩展字段、基础报文 + 业务段落、标准接口 + 客户化字段。JSON Schema 世界也一样。如果你一开始就把对象封得太死，但又没理解组合语义，后面扩展时就会出现“看起来合理，实际上验证不过”的怪问题。

所以今天最有工程价值的结论不是“记住某个关键词”，而是形成这套判断框架：

1. 这个 JSON 边界是开放对象还是封闭对象？
2. 必填约束和字段声明是否已经分开表达清楚？
3. 未来这个 schema 会不会被组合或扩展？
4. 输入校验视角和输出承诺视角是不是同一份 schema？

而 Pydantic 文档把这件事真正落到 Python 上。它告诉你：不要只停留在 `BaseModel` 校验成功这一步，要开始把 `model_json_schema()` / `TypeAdapter.json_schema()` 当成第一等工程产物来看。因为对 Agent 系统来说，真正值钱的不是“某个对象能 parse”，而是“这份边界契约能不能被多个系统共享、验证、复用、演进”。

从转型路径上说，这正是你和很多“只会拼 demo”的人拉开差距的地方。会调模型、会接工具，门槛已经不高；真正难的是把这些调用边界沉淀成长期可维护的契约资产。`JSON Schema` 就是这套资产里的核心形式之一。

## 3 key takeaways

1. `properties` 只定义字段结构，不等于字段必填；`required` 才负责声明哪些字段必须出现。
2. `properties` 也不等于“拒绝未知字段”；JSON Schema 默认允许额外字段，是否关闭要通过 `additionalProperties` 或更适合组合场景的 `unevaluatedProperties` 明确表达。
3. 在 Python Agent 工程里，`BaseModel.model_json_schema()` 和 `TypeAdapter.json_schema()` 产出的不是“顺带生成的文档”，而是可以复用到 tool schema、API、MCP、eval 和状态恢复校验里的统一契约层。

## relation to Agent engineering

这组材料和 Agent 工程的关系，比普通后端接口开发更直接。

第一层关系，是 tool schema 的边界控制。你后面无论是做 OpenAI tool calling、MCP tool，还是自己写 agent host，本质上都需要回答同一个问题：模型到底被允许交出什么参数结构。这里只靠 prompt 文案不够，必须有一层正式 schema。否则系统就会陷入“模型想怎么传都行，执行层再救火”的状态。

第二层关系，是多层复用。Agent 系统里同一份结构往往会同时出现在：

1. 模型输出约束
2. Python 校验模型
3. HTTP 接口定义
4. MCP 消息载荷
5. eval 样本格式
6. 回放与恢复快照

如果每层自己维护一版字段表，后面排查问题会非常痛苦。`JSON Schema` 的价值，就是把这些边界尽量收敛到同一份契约描述上。

第三层关系，是输入视角和输出视角分离。Pydantic 的 `validation` / `serialization` 模式提醒你：系统愿意接受的结构，不一定等于系统最终稳定输出给别人的结构。这一点在 Agent 场景尤其常见，因为模型侧输入常常要更宽容，外部接口和状态持久化侧输出则应该更稳定、更保守。

第四层关系，是你原有 Java / 集成经验的迁移。你过去最有价值的能力之一，不是写类，而是守边界。今天这组内容本质上是在 Python / Agent 世界里给你补齐同一套能力：把边界从“代码里大概长这样”提升到“跨系统都能执行的一致契约”。

## a small action for tonight

今晚做一个 40 分钟的小实验，不用写大功能，只把“统一契约层”真正打通：

1. 用 Pydantic 定义一个 `SearchArgs` 模型，字段是 `query`、`top_k`、`filters`。
2. 打印 `SearchArgs.model_json_schema()`，认真看 `required`、`properties`、`title` 和可能出现的 `$defs`。
3. 手动给模型加一个“你不想放行的额外字段”，然后思考当前 schema 默认会不会拒绝它。
4. 再定义一个 `TypedDict` 版本的 `SearchArgsLite`，用 `TypeAdapter(SearchArgsLite).json_schema()` 对比一下输出差异。
5. 最后把这两个 schema 想象成未来要喂给 tool calling、FastAPI 和 eval 的同一份契约，写下你更愿意在哪些场景用 `BaseModel`，哪些场景用 `TypedDict`。

## 原文关键段落翻译（人工翻译，放在文末）

1. 在 JSON Schema 里，`properties` 用来定义对象上各个属性的 schema；没有匹配到这些属性名的字段，不会被 `properties` 自己处理。
2. 默认情况下，省略某些属性是合法的；换句话说，字段被定义了，不代表它一定必填。
3. 默认情况下，提供额外属性也是合法的；如果希望禁止未声明字段，需要显式使用 `additionalProperties: false`。
4. `additionalProperties` 只识别与自己位于同一个子 schema 中声明的属性；因此在 `allOf` 这类组合场景下，它可能把扩展出来的字段也视为额外字段。
5. `unevaluatedProperties` 和 `additionalProperties` 类似，但它能识别在其他子 schema 中已经成功验证过的属性，因此更适合可组合 schema。
6. Pydantic 里，`BaseModel.model_json_schema()` 与 `TypeAdapter.json_schema()` 返回的是描述结构的 schema 字典，不要和返回实例 JSON 字符串的 `model_dump_json()`、`dump_json()` 混淆。
7. JSON Schema 生成默认使用 `validation` 模式；这个模式描述的是“用于输入校验的 schema”，而不是简单等同于所有输出形态。
8. Pydantic 生成的 JSON Schema 兼容 Draft 2020-12，并可通过 `$defs` 与 `$ref` 组织复用结构。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://json-schema.org/understanding-json-schema/reference/object
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://pydantic.dev/docs/validation/dev/concepts/json_schema/
