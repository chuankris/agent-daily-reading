# 2026-05-26 Agent 工程雷达：工作区运行时、JVM 接入与企业治理开始收敛

## 今日雷达总览（10条）
1. **Google 在 2026 年 5 月 19 日发布 Gemini 3.5，继续把“高速度 + 可执行 agent”绑在一起**
摘要：Google 把 Gemini 3.5 明确定位成适合复杂 agentic workflow 的模型，不再只是强调多模态或长上下文，而是把“action”直接写进主叙事。
为什么值得关注：这说明一线模型竞争点已经转向“推理质量是否能稳定驱动工具调用与长任务执行”。以后看模型，不能只盯 benchmark，要看它是否适合作为 agent runtime 的默认执行引擎。

2. **OpenAI 在 2026 年 4 月 15 日升级 Agents SDK，把 harness、sandbox、snapshot/rehydration 推成标准件**
摘要：新版 Agents SDK 直接覆盖文件检查、命令执行、代码修改和长任务恢复，把很多过去要自己拼的 runtime 能力收进官方框架边界。
为什么值得关注：这对工程实践影响很大。真正难的不是“会不会 prompt”，而是 workspace 怎么挂、状态怎么恢复、失败后如何续跑。官方把这些抽象稳定下来，说明 runtime engineering 正成为主战场。

3. **Anthropic 在 2026 年 5 月 18 日收购 Stainless，继续加码 SDK 与 MCP server tooling**
摘要：Anthropic 公开强调，agent 的能力上限取决于它能连接到多少系统；Stainless 带来的正是 SDK、CLI 与 MCP server 生成能力。
为什么值得关注：这让“连接层”不再只是集成细节，而是平台护城河。对你这种做过系统集成的人，这是直接利好，因为 API 规范、权限边界、协议封装会越来越值钱。

4. **Google 在 2026 年 5 月 21 日发布 ADK for Kotlin 与 ADK for Android 0.1.0**
摘要：Google 把 Agent Development Kit 继续往 JVM 和移动端扩，强调 Kotlin/Android 也可以成为 agent 的一等开发入口。
为什么值得关注：这说明你的 Java/Kotlin 背景不是包袱。企业 agent 不会只活在 Python notebook 里，JVM 生态、移动端入口、本地模型与工具封装都会逐步进入主流程。

5. **OpenAI 在 2026 年 5 月 7 日发布 GPT-Realtime-2、GPT-Realtime-Translate、GPT-Realtime-Whisper**
摘要：OpenAI 这次把实时语音模型升级成更像“持续执行代理”的入口，强调对话中保持上下文、处理中途改口、边说边调用工具。
为什么值得关注：如果后面要做差异化作品，语音 agent 很适合结合设备巡检、工单录入、现场支持这类 IoT 邻近场景，能把你的旧经验转成新的产品切口。

6. **OpenAI 的 `openai-agents-js` 在 2026 年 5 月 5 日发布 v0.9.0，引入 Sandbox Agents**
摘要：这个开源 JavaScript SDK 新增 `SandboxAgent`、Manifest、snapshot、resume、workspace memory 等能力，把受控执行工作区做成了可直接复用的库层能力。
为什么值得关注：这和官方 Agents SDK 的方向高度一致，说明“workspace contract + sandbox + resume”不是一次性产品特性，而是在 SDK 层形成通用工程模式。做个人项目时，这种模式比手写 agent loop 更可复用。

7. **OpenAI 在 2026 年 5 月 8 日公开 Codex 内部安全运行方式，把 approval、边界和 telemetry 讲透**
摘要：这篇工程文不是在讲模型更强，而是在讲如何限制 coding agent 的权限、何时要求人工批准、以及如何保留可审计轨迹。
为什么值得关注：这是非常实战的信号。企业 agent 真正难上线的原因，常常不是准确率，而是安全边界、审批闸门和行为可追踪性。你后续做作品集时，这些比“多 agent 花活”更能体现成熟度。

8. **IBM Research 与 Hugging Face 在 2026 年 5 月 18 日推出 Open Agent Leaderboard**
摘要：这个榜单明确评测的是完整 agent system，而不是只看模型名字；工具集、记忆、规划、恢复和成本都会影响结果。
为什么值得关注：这会让路线讨论更务实。以后谈“哪个 agent 强”，不能只说模型，而要把 scaffold、tooling、memory、recovery 一起说清楚。这对做 eval、trace 和实验对比非常有帮助。

9. **MCP Java SDK 在 2026 年 5 月 21 日发布 v1.1.3 与 v2.0.0-M3，稳定线和下一代线同步推进**
摘要：`modelcontextprotocol/java-sdk` 一边继续维护 1.x 稳定线，一边在 2.0 预发布里推进 JSON Schema 校验、图标元数据支持、SSE transport 退场与更严格的协议实现。
为什么值得关注：这说明 JVM 侧 MCP 工程已经不再停留在“能跑 demo”，而是进入协议细节、兼容性和安全边界持续收敛的阶段。对想从 Java 集成转进 agent 工程的人，这是一条非常现实的桥。

10. **OpenAI 在 2026 年 4 月 28 日把 OpenAI models、Codex 与 Managed Agents 带进 AWS**
摘要：OpenAI 与 AWS 这次不是单纯上模型，而是把模型、编码 agent、受管 agent 一起放进企业熟悉的云治理和采购路径里。
为什么值得关注：这代表 agent 平台化正在进入企业基础设施层。未来很多项目不会从零自建，更多会从受管 runtime 起步，工程重点转向权限、审计、网络边界和系统接入。

## 重点解读（1条）
### MCP Java SDK 进入“稳定可用 + 快速追规格”的阶段，JVM 路线已经足够值得押注

这条最值得你认真看，因为它和你的转型路线贴得最紧。

如果把过去半年 MCP 在 Java 侧的发展放在一起看，一个很清楚的趋势是：**JVM 已经不只是“也能接 MCP”，而是在向正式生产接入层靠拢。**

第一，`v1.1.3` 还在持续做稳定线回补，说明它不是一次性发布后放着不管。像 SSE message endpoint 校验这类修复，看起来不性感，但对真正接企业网络环境、代理层、反向网关的系统非常关键。它说明维护者已经在处理真实部署里会遇到的问题，而不只是示例代码层面的问题。

第二，`v2.0.0-M3` 暴露出的重点也很有代表性：JSON Schema 校验、图标与元数据支持、SSE transport 弃用、同源校验放宽、conformance/security 升级。这些词背后其实是在回答三个问题：

1. 协议会不会继续演进，而且 SDK 能不能跟得上。
2. 复杂企业环境里，transport、安全与兼容性怎么处理。
3. MCP 不再只是“工具调用一下”，而是在往更完整的 agent tool contract 体系扩展。

第三，这对你的现实意义不是“以后都用 Java 写 agent”，而是：
**把 Java 当成企业接入层、协议封装层、权限治理层，再把 Python/TypeScript 当成实验和上层编排层。**

这条路线比“彻底抛掉 Java，全栈重学 Python”更稳，也更符合企业项目实际。你原来做过的设备接口、外部系统接入、异常处理、审计日志、重试补偿、权限隔离，这些都能自然迁移到 MCP server/client、tool wrapper、approval gate、trace pipeline 这些 agent 工程核心部件上。

一句话判断：
**对你现在最优的转型姿势，不是站队某个框架，而是尽快把“Java 集成能力 -> MCP 接入能力 -> Agent 工具能力”这条桥亲手搭出来。**

## 对当前转型路线的影响
- 接下来优先补的不是更多框架名字，而是 `workspace manifest`、`sandbox/runtime`、`resume`、`tool contract`、`approval/telemetry` 这五层。
- 技术栈上不必做非黑即白切换。更现实的分工是：Python/TypeScript 负责 agent 编排与实验，Java/Kotlin 负责 MCP 接入、业务系统封装与稳定执行。
- 作品集选题上，优先做“能接真实系统、可恢复、可审计、可审批”的单 agent 闭环，比表面复杂的多 agent demo 更有说服力。
- 如果你想把 IoT 背景转成优势，后续最值得做的是“语音入口 + MCP 工具 + 设备/工单系统 + 人工审批回写”这一类场景。

## 今晚可验证动作（10-20分钟）
1. 打开 MCP Java SDK 的 release 页面，分别看 `v1.1.3` 和 `v2.0.0-M3`，整理一张两列表：`稳定维护项` / `协议演进项`。
2. 选一个你熟悉的旧系统动作，比如“查询设备状态”或“创建工单”，写一个最小 MCP tool contract：输入、输出、权限、超时、失败码。
3. 用 5 分钟画出你自己的双层架构草图：`Java MCP 接入层 -> Python/TS Agent 编排层 -> 审计/批准/追踪`。

## 原文链接
- 1. https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-3-5/
- 2. https://openai.com/index/the-next-evolution-of-the-agents-sdk/
- 3. https://www.anthropic.com/news/anthropic-acquires-stainless
- 4. https://developers.googleblog.com/adk-kotlin-android-building-ai-agents/
- 5. https://openai.com/index/advancing-voice-intelligence-with-new-models-in-the-api/
- 6. https://github.com/openai/openai-agents-js/releases/tag/v0.9.0
- 7. https://openai.com/index/running-codex-safely/
- 8. https://huggingface.co/blog/ibm-research/open-agent-leaderboard
- 9. https://github.com/modelcontextprotocol/java-sdk/releases
- 10. https://openai.com/index/openai-on-aws

## 原文中文翻译链接（机器翻译）
- 1. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-3-5/
- 2. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/the-next-evolution-of-the-agents-sdk/
- 3. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://www.anthropic.com/news/anthropic-acquires-stainless
- 4. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://developers.googleblog.com/adk-kotlin-android-building-ai-agents/
- 5. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/advancing-voice-intelligence-with-new-models-in-the-api/
- 6. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/openai/openai-agents-js/releases/tag/v0.9.0
- 7. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/running-codex-safely/
- 8. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://huggingface.co/blog/ibm-research/open-agent-leaderboard
- 9. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/modelcontextprotocol/java-sdk/releases
- 10. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/openai-on-aws
