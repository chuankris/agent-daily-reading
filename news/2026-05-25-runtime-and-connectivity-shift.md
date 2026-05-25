# 2026-05-25 Agent 工程雷达：运行时、连接层、多语言 SDK 正在一起成形

## 今日雷达总览（10条）
1. **Google 在 2026 年 5 月 19 日发布 Gemini 3.5 Flash，并把 Managed Agents 与 Antigravity 2.0 一起推到前台**
摘要：Google 在 I/O 2026 把重点从“更强模型”进一步推进到“可直接干活的 agent 平台”，同一波里同时给出 Gemini 3.5 Flash、Antigravity 2.0、Gemini API Managed Agents、AI Studio 到 Android 的连通路径。
为什么值得关注：这说明头部厂商竞争点已经不是单个模型分数，而是“模型 + 运行时 + 状态恢复 + 多端开发入口”的整套交付能力。以后选型要更重视 agent harness、持久会话和部署路径。

2. **OpenAI 在 2026 年 4 月 15 日升级 Agents SDK，把 sandbox、manifest、snapshot/rehydration 做成一等能力**
摘要：新版 Agents SDK 不再只是多 agent 编排库，而是直接给出模型原生 harness、内置 sandbox 执行、Manifest 工作区描述，以及运行失败后的快照恢复能力。
为什么值得关注：这条非常工程化。Agent 系统最难的往往不是 prompt，而是“文件怎么挂载、命令在哪跑、状态怎么接着跑、子 agent 怎么隔离”。官方把这些抽到 SDK 层，意味着 runtime engineering 正在标准化。

3. **Anthropic 在 2026 年 4 月 16 日发布 Claude Opus 4.7，并明确把它当作更强 agent/coding/vision 工作模型**
摘要：Anthropic 把 Opus 4.7 定位为比 Opus 4.6 更强的多步任务与专业工作模型，同时把网络安全高风险请求的自动阻断作为发布的一部分。
为什么值得关注：这说明前沿模型发布已经和“安全上线机制”绑在一起。做企业 agent 时，模型能力和策略护栏不该分开看，尤其是会调用真实工具的系统。

4. **Google 在 2026 年 5 月 21 日发布 ADK for Kotlin 与 ADK for Android 0.1.0**
摘要：继 Java、Go 1.0 和 Python 2.0 beta 后，Google 又把 ADK 扩展到 Kotlin 与 Android，还强调 Android 侧可结合本地模型做 on-device agent。
为什么值得关注：这对 Java/Kotlin 背景工程师是直接利好。你原来的集成、接口封装、异常处理、审批闸门经验，可以更自然地迁移到 JVM 侧 agent 工程，而不是被迫全部换到纯 Python。

5. **OpenAI 在 2026 年 5 月 7 日发布 GPT-Realtime-2、GPT-Realtime-Translate、GPT-Realtime-Whisper**
摘要：这一代实时语音模型从“会说话”升级到“边对话边推理、翻译、转写并调用工具”，把语音 agent 往可交付产品推进了一步。
为什么值得关注：如果你后面想做差异化作品，语音入口非常适合结合现场运维、设备巡检、工单处理这类场景。IoT 背景与实时语音 agent 的贴合度很高。

6. **OpenAI 在 2026 年 4 月 28 日宣布 OpenAI models、Codex、Managed Agents 进入 AWS**
摘要：OpenAI 与 AWS 把模型、Codex 与 Bedrock Managed Agents 放进同一企业云采购和治理路径中，主打 limited preview 下的安全、合规、采购流程兼容。
为什么值得关注：这代表 agent 平台化正在进入企业基础设施层。以后很多落地项目不一定从“自建全部基础设施”开始，而是从云厂商提供的受管运行时开始，工程重点会转向权限、审计和系统接入。

7. **Anthropic 在 2026 年 5 月 18 日收购 Stainless，把 SDK 生成与 MCP server tooling 纳入平台能力**
摘要：Anthropic 明说“agent 的能力取决于它能连接到什么系统”，并点名 Stainless 在 SDK、CLI、MCP server tooling 上的积累。
为什么值得关注：这强化了一个很现实的判断: 2026 年 agent 工程的瓶颈之一就是连接层。API 规范、SDK 生成、MCP server、权限封装，这些“脏活累活”正在成为平台护城河。

8. **IBM Research 与 Hugging Face 在 2026 年 5 月 18 日推出 Open Agent Leaderboard**
摘要：这个榜单不再只比单模型，而是把 agent + model + benchmark 当成完整系统评测，还统一了不同 benchmark 的协议结构。
为什么值得关注：技术路线争论会更务实。以后谈“哪个 agent 更强”，必须同时问清楚 scaffold、工具集、记忆、恢复机制与成本，而不是只看一个模型名。

9. **MCP Java SDK 最新稳定版在 2026 年 4 月 25 日来到 v1.1.2，并明确与 Spring AI 协作维护**
摘要：`modelcontextprotocol/java-sdk` 已有稳定发布节奏，仓库 README 也明确写出它是官方 Java SDK，并与 Spring AI 协作维护。
为什么值得关注：这就是你现有技术栈通向 agent 工程的桥。比起空泛地“转 Python”，更实际的路线是先把旧系统能力包装成 MCP server/client，再把它接进更大的 agent 体系。

10. **Cohere 在 2026 年 5 月 19 日收购 Reliant AI，继续押注 sovereign enterprise AI**
摘要：Cohere 把主权部署、受监管行业、垂直领域数据与模型能力捆到一起，重点落在医疗和生命科学等高合规环境。
为什么值得关注：这提醒你别把 agent 只理解成通用办公助手。真正高价值场景往往在“数据敏感 + 流程复杂 + 需要系统接入”的行业里，这和企业集成背景更对路。

## 重点解读（1条）
### OpenAI Agents SDK 的这次升级，本质上是在把 Agent 工程从“编排 prompt”推向“设计运行时”

如果只看表面，这次更新像是在 SDK 里多加了几个能力；但从工程视角看，它其实在重新定义 agent 框架该负责什么。

第一层变化，是 **运行环境被正式纳入框架边界**。过去很多 agent 项目卡在“模型会计划，但不会稳定执行”：文件在哪、命令怎么跑、依赖怎么隔离、失败后怎么接着跑，都要团队自己补。现在 harness + sandbox 成为一等能力，意味着官方默认你做的不是问答机器人，而是会接触真实文件和真实工具的系统。

第二层变化，是 **状态恢复开始标准化**。snapshot/rehydration 这类词很像基础设施语言，不像 AI demo 语言。它背后的含义是：一次 agent run 可以被当成长任务来运营，而不是一次性推理。对生产系统来说，这比“再多 2 分 benchmark”更重要，因为真正烧时间的是中断、超时、环境丢失和人工接管。

第三层变化，是 **Manifest 把工作区契约显式化**。这件事对工程团队尤其关键。你不再只是给 agent 一段 instructions，而是明确描述输入文件、输出目录、对象存储挂载和隔离边界。长期看，这会让 agent 项目更像可审计的作业系统，而不是不可控的黑盒 prompt。

第四层变化，是 **多 agent 的价值开始从“多角色对话”回到“多隔离执行单元”**。官方强调子 agent 可以路由到独立 sandbox、按需拉起环境、跨容器并行。也就是说，多 agent 最有价值的地方不一定是让它们互相辩论，而是把不同风险级别、不同工具权限、不同计算负载拆开管理。

一句话判断：
**2026 年更值得学习的，不是再多背几种 agent pattern 名字，而是把 runtime、state、tool contract、approval、trace 这几层真正做扎实。**

## 对当前转型路线的影响
- 你接下来一周最该补的不是更多框架名词，而是 `tool contract`、`sandbox/runtime`、`state resume`、`trace/eval`、`approval gate` 这五件事。
- 作品集方向上，优先做“能接真实系统、能失败恢复、能人工接管”的单 agent 闭环，比做表面复杂的多 agent demo 更有说服力。
- Java/Kotlin 不该被当成历史包袱。结合 MCP Java SDK、ADK for Kotlin、Spring AI，这反而是你进入企业 agent 集成层的现实优势。
- 如果要体现 IoT/集成背景，后续选题可以优先考虑“语音入口 + 设备/工单系统 + agent 工具调用 + 审批回写”这条链路。

## 今晚可验证动作（10-20分钟）
1. 读 OpenAI Agents SDK 新文里关于 `Manifest`、`snapshotting`、`rehydration` 的段落，写出你自己的 3 句话版本定义。
2. 选一个你熟悉的旧系统动作，比如“查询设备状态”或“下发远程命令”，补一版 agent tool contract：输入、输出、超时、重试、人工审批条件。
3. 用 5 分钟画一个最小运行时图：`User -> Planner Agent -> Tool Wrapper/MCP -> Business System -> Trace/Audit -> Human Approval`。

## 原文链接
- 1. https://blog.google/innovation-and-ai/technology/developers-tools/google-io-2026-developer-highlights/
- 2. https://openai.com/index/the-next-evolution-of-the-agents-sdk/
- 3. https://www.anthropic.com/news/claude-opus-4-7
- 4. https://developers.googleblog.com/en/adk-kotlin-android-building-ai-agents/
- 5. https://openai.com/index/advancing-voice-intelligence-with-new-models-in-the-api/
- 6. https://openai.com/index/openai-on-aws
- 7. https://www.anthropic.com/news/anthropic-acquires-stainless
- 8. https://huggingface.co/blog/ibm-research/open-agent-leaderboard
- 9. https://github.com/modelcontextprotocol/java-sdk
- 10. https://cohere.com/blog/cohere-acquires-reliant-ai-expand-sovereign-enterprise-ai

## 原文中文翻译链接（机器翻译）
- 1. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://blog.google/innovation-and-ai/technology/developers-tools/google-io-2026-developer-highlights/
- 2. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/the-next-evolution-of-the-agents-sdk/
- 3. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://www.anthropic.com/news/claude-opus-4-7
- 4. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://developers.googleblog.com/en/adk-kotlin-android-building-ai-agents/
- 5. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/advancing-voice-intelligence-with-new-models-in-the-api/
- 6. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/openai-on-aws
- 7. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://www.anthropic.com/news/anthropic-acquires-stainless
- 8. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://huggingface.co/blog/ibm-research/open-agent-leaderboard
- 9. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/modelcontextprotocol/java-sdk
- 10. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://cohere.com/blog/cohere-acquires-reliant-ai-expand-sovereign-enterprise-ai
