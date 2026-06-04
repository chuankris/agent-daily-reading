# 2026-06-03 Agent 工程雷达：云入口与托管 Agent 栈收敛
## 今日雷达总览（10条）
1. **OpenAI 于 2026 年 6 月 1 日宣布 frontier models 与 Codex 在 AWS 正式可用**
摘要：OpenAI 把模型与 Codex 一起放进 AWS 路径，支持通过 Amazon Bedrock 和 AWS 现有治理体系接入。  
为什么值得看：这不是单纯“多一个云渠道”，而是把采购、合规、安全评审、计费和上线流程一起打通。对 Agent 工程来说，真正的门槛正在从“会不会调模型”转向“能不能进企业现有控制面”。

2. **Google 于 2026 年 5 月 19 日推出 Gemini API Managed Agents**
摘要：Gemini API 现在支持托管 Agent，一次调用即可拉起远程 Linux 环境，支持推理、调工具、执行代码与网页访问，并可用 `AGENTS.md` / `SKILL.md` 定义自定义能力。  
为什么值得看：这说明“Agent 运行时”正在产品化。你以后更值钱的能力，不是再手写一套 loop，而是会不会把权限、状态、工具、审计和恢复机制接到托管执行面上。

3. **OpenAI 于 2026 年 5 月 28 日更新 GPT-5.5 Instant，并公布 ChatGPT 侧 o3 / GPT-4.5 下线时间**
摘要：GPT-5.5 Instant 在 ChatGPT 和 API 侧更新了风格与质量；同时 OpenAI 宣布 ChatGPT 中的 o3 将于 2026 年 8 月 26 日退役，GPT-4.5 将于 2026 年 6 月 27 日退役，且这些变更不影响 API。  
为什么值得看：模型组合正在快速收敛，默认模型行为也会持续变化。做 Agent 时不能把“今天可用的默认模型”当稳定基础设施，必须把模型选择、回退和评测独立出来。

4. **Google 于 2026 年 5 月 5 日扩展 Gemini API File Search，加入多模态检索、元数据和页级引用**
摘要：File Search 现在不仅能处理文本，也能处理图像，并支持自定义 metadata 和 page-level citations。  
为什么值得看：这对 RAG 很关键。以后可验证性不再只是“检索到了”，而是“能不能指到 PDF 第几页、图像哪个来源、用什么标签过滤出来”。

5. **Google 于 2026 年 5 月 4 日给 Gemini API 加入 Webhooks**
摘要：Google 为长时任务引入事件驱动 Webhooks，用推送替代持续轮询。  
为什么值得看：这直接影响 Agent 系统的工程形态。长任务、批处理、异步审批、外部工具编排，都会从“轮询式”往“事件式”迁移，更接近成熟集成系统。

6. **OpenAI Agents SDK 于 2026 年 5 月 26 日发布 v0.17.4**
摘要：这一版继续补 Realtime、恢复机制、MCP SSE 传输加固，以及 tracing 相关类型导出。  
为什么值得看：重点已经很明确了，官方在补的是恢复、传输稳健性、可观测性，而不是只堆 demo。你做作品集时也该把这些维度放进主线。

7. **Google ADK Python 于 2026 年 5 月 23 日发布 v2.1.0**
摘要：新版本加入 sandbox templates / snapshots、data agent 图表生成与 artifact 加载、更多 telemetry 字段，并修复 MCP tool error 导致 session 掉线的问题。  
为什么值得看：这说明 Agent SDK 的竞争正在从“会不会多 Agent”转向“有没有稳定沙箱、产物管理、遥测和容错能力”。

8. **Google ADK Java 于 2026 年 5 月 15 日发布 v1.3.0**
摘要：Google 的 Java 版 ADK 继续推进，说明官方已经在认真补 JVM 入口，而不是只把 Python 当唯一主战场。  
为什么值得看：这和你的背景直接相关。Java/Kotlin 在企业内更适合做工具接入层、受控 Agent 服务、审批边界和遗留系统封装，不必把转型理解成“只能全栈 Python”。

9. **MCP Java SDK 在 2026 年 5 月进入 2.x 里程碑阶段**
摘要：`2.0.0-M1` 强调 schema 前后向兼容、规范一致性、校验增强和 transport 改进，之后在 5 月下旬继续推进 `2.0.0-M3`。  
为什么值得看：这代表 MCP 正从“能跑”进入“能长期演进”的阶段。对企业集成尤其重要，因为 schema 演进、传输兼容和校验边界，决定了协议是否能真正承载生产接入。

10. **MCP C# SDK 于 2026 年 5 月 8 日发布 v1.3.0**
摘要：这一版重点补 transport 诊断能力与稳定性，让客户端能结构化获取退出码、stderr 尾部和 HTTP 状态，而不是靠字符串猜错误。  
为什么值得看：这类改动看起来不炫，但很生产。Agent 平台一旦进入企业网络、桌面环境或混合部署，排障与诊断信息往往比“再多一个 fancy feature”更值钱。

## 重点解读（1条）
### OpenAI on AWS 值得重点跟，因为它把 Agent 工程的竞争点从“模型接入”推到了“企业落地通道”

OpenAI 在 2026 年 6 月 1 日宣布 frontier models 与 Codex 在 AWS 正式可用，核心不只是模型上架，而是明确押注企业最熟悉的那套落地路径：安全、治理、采购、计费、合规、生产发布都尽量复用 AWS 现有体系。

这背后至少有三层信号：

1. **模型能力开始依附云控制面，而不是只依附模型 API 本身**  
过去大家讨论更多是“哪个模型更强”。现在更现实的问题变成：能否放进企业已经批准的云环境、网络边界、审计流程和成本中心。谁先进入这条路径，谁就更容易从 POC 走到真部署。

2. **Codex 这类软件工程 Agent 被明确包装成企业生产力能力，而不是个人玩具**  
OpenAI 没有只讲聊天能力，而是直接把写代码、审代码、调试、现代化改造放进 AWS 语境里。这对转型做 Agent 工程的人是很强的信号：代码代理正在被当作正式基础设施采购，而不是单点工具。

3. **你的工程重心要继续向“接入与治理”倾斜**  
如果模型能力越来越容易买到，差异化就会转移到接入层：权限模型、工具封装、审计轨迹、异步执行、回滚、观测、成本控制、故障恢复。你过去做 Java/IoT 集成时积累的很多能力，反而会在这里重新变得值钱。

对你最直接的启发不是“马上迁移到 AWS”，而是先建立一个判断框架：  
- 这个 Agent 能不能接进现有 IAM / 审计 / 网络边界？  
- 这个工具调用链出错时，能不能回放、定位、恢复？  
- 这个模型切换后，行为和成本是否可评估？  
- 这个系统是否可以被企业安全团队理解和接管？

如果这四个问题都答不出来，系统大概率还停留在 demo 阶段。

## 对当前转型路线的影响
- 学习重点继续从“Prompt 技巧”转向“Agent 基础设施”：MCP、托管执行面、异步事件、可观测性、权限与审计。
- Java/Kotlin 不该被放弃，应该放到更合适的位置：企业工具封装、MCP server、审批/网关边界、稳定接入层。
- 作品集优先做“能进生产边界”的案例，而不是只做多 Agent 展示。比如：带审计日志的工单 Agent、带恢复机制的批处理 Agent、带页级引用的企业知识 RAG。
- 评估新框架时优先问五件事：状态怎么存、工具怎么控、故障怎么恢复、trace 怎么看、如何接企业身份与网络边界。

## 今晚可验证动作（10-20分钟）
1. 选一个你熟悉的内部系统场景，比如设备告警工单或知识库检索。
2. 只画一页最小架构：`Agent -> Tool Gateway -> Existing API / DB`。
3. 强制补上 6 个工程字段：`request_id`、`user_scope`、`timeout_ms`、`idempotency_key`、`audit_key`、`trace_id`。
4. 再写一条失败路径：工具超时后怎么重试、怎么落审计、怎么让人工接管。
5. 最后判断一次：这个设计如果放进 AWS / 企业网关 / MCP server，哪里最先出问题。

## 原文链接
- 1. https://openai.com/index/openai-frontier-models-and-codex-are-now-available-on-aws/
- 2. https://blog.google/innovation-and-ai/technology/developers-tools/managed-agents-gemini-api/
- 3. https://help.openai.com/en/articles/9624314-model-release-notes
- 4. https://blog.google/innovation-and-ai/technology/developers-tools/expanded-gemini-api-file-search-multimodal-rag/
- 5. https://blog.google/innovation-and-ai/technology/developers-tools/event-driven-webhooks/
- 6. https://github.com/openai/openai-agents-python/releases/tag/v0.17.4
- 7. https://github.com/google/adk-python/releases/tag/v2.1.0
- 8. https://github.com/google/adk-java/releases/tag/v1.3.0
- 9. https://github.com/modelcontextprotocol/java-sdk/releases/tag/v2.0.0-M1
- 10. https://github.com/modelcontextprotocol/csharp-sdk/releases/tag/v1.3.0

## 原文中文翻译链接（机器翻译）
- 1. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/openai-frontier-models-and-codex-are-now-available-on-aws/
- 2. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://blog.google/innovation-and-ai/technology/developers-tools/managed-agents-gemini-api/
- 3. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://help.openai.com/en/articles/9624314-model-release-notes
- 4. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://blog.google/innovation-and-ai/technology/developers-tools/expanded-gemini-api-file-search-multimodal-rag/
- 5. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://blog.google/innovation-and-ai/technology/developers-tools/event-driven-webhooks/
- 6. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/openai/openai-agents-python/releases/tag/v0.17.4
- 7. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/google/adk-python/releases/tag/v2.1.0
- 8. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/google/adk-java/releases/tag/v1.3.0
- 9. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/modelcontextprotocol/java-sdk/releases/tag/v2.0.0-M1
- 10. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/modelcontextprotocol/csharp-sdk/releases/tag/v1.3.0
