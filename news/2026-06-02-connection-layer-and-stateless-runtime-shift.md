# 2026-06-02 Agent 工程雷达：连接层与无状态运行时开始定型

## 今日雷达主题
这两周最该盯的，不只是模型本身，而是 Agent 的“连接层”正在快速定型：协议往无状态走，SDK 往多语言和企业网关靠，托管运行时则把执行、工具和状态管理继续产品化。

## 今日雷达总览（10条）
1. **MCP 于 2026 年 5 月 11 日接受 SEP-2575：Make MCP Stateless**
摘要：MCP 官方接受“无状态优先”方案，目标是去掉强绑定初始化握手，让每个请求都能独立处理，更容易放到标准负载均衡和云原生部署里。
为什么重要：这不是小修小补，而是在改 Agent 连接层的默认形状。对你这种做过集成的人来说，这意味着 MCP server 更像标准 API 服务，而不是脆弱的长连接会话程序。

2. **Anthropic 于 2026 年 5 月 18 日收购 Stainless**
摘要：Anthropic 直接把做 SDK、CLI 和 MCP server 生成链路的 Stainless 收进来，继续强化 Claude 的开发者连接层。
为什么重要：大模型公司开始把“接口到工具接入”当核心资产，而不是外围生态。以后谁能把 API spec、SDK、MCP、权限和错误处理做顺，谁就更容易吃下企业 Agent 落地。

3. **Google 于 2026 年 5 月 21 日发布 ADK for Kotlin 与 ADK for Android 0.1.0**
摘要：Google 把 ADK 正式扩到 Kotlin 和 Android，补上 JVM 后端和端侧 Agent 的官方入口。
为什么重要：这对你的背景高度相关。它说明 Java/Kotlin 不是被 Agent 时代抛下的语言，而是在企业后端、移动端和设备侧继续获得正式支持。

4. **Google 于 2026 年 5 月 19 日推出 Gemini API Managed Agents**
摘要：Gemini API 现在能单次调用拉起会推理、调工具、执行代码的托管 Agent，且运行在可续接的隔离 Linux 环境里。
为什么重要：底层 runtime 越来越像云服务能力。今后更有价值的工作，不是再造 loop，而是把真实系统、审批、审计和恢复策略接进托管执行面。

5. **OpenAI 于 2026 年 5 月 7 日发布 GPT-Realtime-2、GPT-Realtime-Translate 与 GPT-Realtime-Whisper**
摘要：OpenAI 在 API 中推出新一代实时语音模型，覆盖推理型语音对话、实时翻译和低延迟转写。
为什么重要：这说明“Agent = 文本”这层假设正在失效。后续很多现场运维、客服、设备协作和语音工单场景，会直接把语音作为 Agent 的一等输入面。

6. **OpenAI Agents SDK 于 2026 年 5 月 26 日发布 v0.17.4**
摘要：这版继续补 Realtime、自恢复、MCP SSE 传输加固和 tracing 类型导出，明显在往更稳的生产工程链路收口。
为什么重要：OpenAI 的 Agent SDK 更新节奏说明一件事：真正难点已经不是“能不能调模型”，而是 Realtime、恢复、传输安全和可观测性这些细节能不能扛住长任务。

7. **MCP Java SDK 于 2026 年 5 月 21 日发布 v2.0.0-M3**
摘要：Java SDK 继续推进 2.x 线，围绕协议兼容、可扩展 schema 和更稳的企业集成能力演进。
为什么重要：如果你想把 Java 放在企业工具封装层、MCP server 或稳定接入层，这条线值得长期跟。它在把 JVM 侧的 MCP 工程化补齐，而不是停留在 demo 水平。

8. **MCP C# SDK 于 2026 年 5 月 8 日发布 v1.3.0**
摘要：官方 C# SDK 持续迭代，仓库也明确把 HTTP server、AspNetCore 集成和轻量 core 包拆分出来。
为什么重要：这再次验证 MCP 正在朝主流企业语言收敛。对跨系统、跨部门落地而言，协议一旦在 Java 和 .NET 都站住，企业导入阻力会明显下降。

9. **Microsoft Agent Framework 于 2026 年 5 月 7 日发布 1.3.0**
摘要：新版本增加类式 skill 定义、实验性的 session/todo/memory harness context provider、prompt injection defense，以及对 OpenAI/Gemini `allowed_tools` 的支持。
为什么重要：这反映微软也在把 Agent 框架往“可治理的执行容器”推进，而不是只堆多 Agent 编排语法。你看框架时要多问治理和防注入，不要只看 orchestration 演示。

10. **Open Multi-Agent 于 2026 年 5 月 9 日发布 v1.4.0**
摘要：这个 TypeScript 原生开源项目主打“从目标自动拆到任务 DAG”，强调低依赖、多 Agent 编排、MCP 和 live tracing。
为什么重要：它代表一个很现实的路线分化：上层编排开始更强调 DAG、并行和 trace，而不是简单串行 loop。你做作品集时，可以借它对照“任务图”思维而不是只写单 agent。

## 重点解读（1条）
### MCP 的“无状态优先”值得重点跟，因为它最像企业级 Agent 基建真正会采用的形状

SEP-2575 的核心不是一句“stateless 更先进”，而是把 MCP 从“依赖会话握手的协议”往“能被标准基础设施托住的协议”移动。

这背后有三个非常现实的工程收益：

1. **更容易水平扩展**
如果每个请求都自描述，MCP server 就更容易挂到普通负载均衡、API gateway、服务网格后面，不必强依赖 sticky session。对企业来说，这会直接影响成本、弹性和故障恢复。

2. **更接近你熟悉的集成模式**
你做过 Java / IoT 集成，应该很熟悉“签名、鉴权、超时、幂等、重试、审计”这一套。无状态 MCP 让 Agent 工具层更容易复用这些成熟做法，而不是另起一套特殊长连接心智。

3. **把复杂度从协议默认项，降成按需能力**
官方这次的方向，本质上是在做“默认简单，必要时再加状态”。这对生态很重要，因为只有默认模型足够简单，Java、.NET、网关、中台和安全团队才更容易接受。

对你的转型路线，最直接的启发不是“马上去实现完整 MCP 协议”，而是先建立一个判断标准：

- 这个工具接入如果必须靠长连接会话才能工作，能不能拆成显式状态句柄或自包含请求？
- 这个 Agent 任务如果断线或重试，状态是不是还能恢复？
- 这个工具调用如果经过网关、审计和权限系统，会不会比普通 API 特殊太多？

如果答案越来越偏向“和普通企业 API 更像”，那就说明这个方向更适合真正落地，而不是只适合演示。

## 对当前转型路线的影响
- 学习重点应继续从“prompt 技巧”向“连接层工程”倾斜：协议、鉴权、trace、恢复、网关适配。
- Java/Kotlin 的价值在上升：很适合做 MCP server、企业接口包装、审批边界和稳定工具层。
- 作品集建议优先做“有状态业务，走无状态接入”的案例，比如工单、设备查询、知识检索、审批工具。
- 评估框架时优先问四件事：是否支持标准网关、是否便于审计、是否支持恢复、是否便于多语言接入。

## 今晚可验证动作（10-20分钟）
1. 选一个你熟悉的内部接口，写出它被包装成 MCP tool 后的最小输入输出 schema。
2. 再补 5 个工程字段：`request_id`、`auth_scope`、`timeout_ms`、`idempotency_key`、`audit_key`。
3. 最后判断一次：这个 tool 能否在不依赖粘性会话的前提下完成一次调用；如果不能，缺的状态句柄是什么。

## 原文链接
- 1. https://modelcontextprotocol.io/seps/2575-stateless-mcp
- 2. https://www.anthropic.com/news/anthropic-acquires-stainless
- 3. https://developers.googleblog.com/adk-kotlin-android-building-ai-agents/
- 4. https://blog.google/innovation-and-ai/technology/developers-tools/google-io-2026-developer-highlights/
- 5. https://openai.com/index/advancing-voice-intelligence-with-new-models-in-the-api/
- 6. https://github.com/openai/openai-agents-python/releases/tag/v0.17.4
- 7. https://github.com/modelcontextprotocol/java-sdk/releases/tag/v2.0.0-M3
- 8. https://github.com/modelcontextprotocol/csharp-sdk/releases/tag/v1.3.0
- 9. https://github.com/microsoft/agent-framework/releases/tag/python-1.3.0
- 10. https://github.com/open-multi-agent/open-multi-agent/releases/tag/v1.4.0

## 原文中文翻译链接（机器翻译）
- 1. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://modelcontextprotocol.io/seps/2575-stateless-mcp
- 2. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://www.anthropic.com/news/anthropic-acquires-stainless
- 3. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://developers.googleblog.com/adk-kotlin-android-building-ai-agents/
- 4. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://blog.google/innovation-and-ai/technology/developers-tools/google-io-2026-developer-highlights/
- 5. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/advancing-voice-intelligence-with-new-models-in-the-api/
- 6. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/openai/openai-agents-python/releases/tag/v0.17.4
- 7. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/modelcontextprotocol/java-sdk/releases/tag/v2.0.0-M3
- 8. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/modelcontextprotocol/csharp-sdk/releases/tag/v1.3.0
- 9. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/microsoft/agent-framework/releases/tag/python-1.3.0
- 10. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/open-multi-agent/open-multi-agent/releases/tag/v1.4.0
