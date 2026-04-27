# 2026-04-27 为什么 Agent 早期就该用 Structured Outputs

原文来源：

- 标题：Structured model outputs | OpenAI API
- 链接：https://developers.openai.com/api/docs/guides/structured-outputs
- 来源类型：官方文档
- 发布时间：页面未单独标注

为什么今天读这个：

你现在已经知道“模型可以调工具”，但这还不够。真正进入 Agent 工程后，问题会很快从“能不能回答”变成“输出能不能稳定进入下一步系统”。对有长期 Java/系统集成背景的人，这个区别很关键，因为你的强项本来就不是聊天效果，而是把系统之间的输入输出边界收紧。

如果继续只靠自由文本和提示词约束，后面你做这些事情都会很痛苦：

1. 把模型输出直接喂给后端接口。
2. 把结果存进数据库或消息队列。
3. 把一个 Agent 的输出交给下一个 Agent 或 workflow 节点。
4. 做自动评测、回放和失败重试。

中文精读：

这篇官方文档的重点，不是“让 JSON 更漂亮”，而是把模型输出从“给人看”推进到“给程序消费”。

很多初学者会把 JSON mode 和 Structured Outputs 混在一起。它们的差别非常像你做系统集成时的两种接口状态：

1. JSON mode
- 更像“对方承诺给你一个 JSON 字符串”。
- 但字段是否完整、枚举值是否有效、层级是否符合你预期，并没有被真正约束住。

2. Structured Outputs
- 更像“双方先约定一个 schema，再按 schema 严格交付”。
- 你的程序可以更稳定地解析、校验和路由结果。

这件事对 Agent 工程的意义非常大。因为 Agent 不是单次问答，而是一个会继续驱动工具、状态机、评测器、数据库和下游服务的系统。只要输出不稳定，系统复杂度会立刻转移到：

- 重试逻辑
- 容错解析
- 脏数据清洗
- 边界 case 修补

文档里有几个点尤其值得你用工程化思维记住。

第一，schema 本身就是接口契约，不只是格式提示。文档强调使用 `strict: true`，并且对象里要显式限制 `additionalProperties: false`。这背后的思想非常像你定义 Java DTO、OpenAPI schema、消息协议时的原则：系统不应该默默接受多余字段，更不应该指望调用方“自己懂”。

第二，key 的命名、描述、结构设计本身会影响生成质量。也就是说，Structured Outputs 不是“有 schema 就完事”，而是要把 schema 当成 prompt 的一部分来设计。对你来说，这非常像集成项目里的接口设计经验：一个字段叫 `status` 还是 `next_action`，一个数组元素是 `step` 还是 `tool_call_result`，会直接影响下游系统理解成本，也会影响模型命中正确结构的稳定性。

第三，官方文档明确建议把 schema 设计和 eval 结合起来看。这一点很值得重视，因为你昨天已经补了 evaluation best practices。正确做法不是“先拍脑袋定一个 schema，以后不动”，而是：

1. 先定义一个最小可用结构。
2. 用真实样例跑起来。
3. 统计哪些字段经常错、漏、歧义大。
4. 再迭代 schema 和描述。

这套方式和你过去做 ToB 交付很像。客户接口经常不是一次定完，而是通过联调、脏数据、边界场景逐渐收敛。Structured Outputs 也一样，只不过这次对接方变成了模型。

第四，Structured Outputs 会让多步 Agent 设计更干净。比如你后面做任务拆解 Agent，不要让它输出一大段自然语言再让程序从中抠字段，而应该直接要求它生成类似下面的结构：

- `task_type`
- `need_tools`
- `risk_level`
- `next_step`
- `final_answer`

这样你的 workflow、日志、评测、失败重放都更容易做。你会明显感觉到，它更接近“状态机节点输出”，而不是“聊天记录”。

如果继续用 Java/后端经验来翻译，这篇文档的本质可以理解成：

1. Prompt 负责意图约束。
2. Schema 负责接口契约。
3. Eval 负责验证契约在真实样本上是否稳定成立。

这三者合在一起，才是 Agent 工程里的“结构化交付”。

3 个关键收获：

1. JSON mode 只保证像 JSON，Structured Outputs 才更接近真正的接口契约。
2. `strict: true` 和清晰 schema 设计，能显著降低下游解析、重试和脏数据成本。
3. 输出 schema 也要进入 eval 闭环，不能只靠一次 prompt 调通。

和 Agent 工程的关系：

你后面无论做 tool calling、任务拆解、RAG 回答打分、还是多 Agent handoff，本质上都在处理“一个节点输出，另一个节点消费”。Structured Outputs 能把这个边界提前收紧，让 Agent 系统更像工程系统，而不是拼运气的聊天脚本。

今晚可动手：

用你熟悉的“接口定义先行”思路，给一个最小 Agent 结果先写出 schema 草稿，例如：

1. `intent`
2. `answer`
3. `need_tool`
4. `tool_name`
5. `confidence`

然后问自己两个问题：

1. 哪些字段是必须的？
2. 哪些字段如果多出来，后端应该直接拒绝？
