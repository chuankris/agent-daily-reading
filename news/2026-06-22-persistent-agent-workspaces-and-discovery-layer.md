# 2026-06-22 Agent 工程雷达：持久化工作区与发现层前移

## 今日雷达总览（10条）

1. **OpenAI 在 2026-06-16 发布 Deployment Simulation，把“上线前评估”推进到真实对话重放**
摘要：OpenAI 用去标识化历史对话重放新模型输出，并且已经把方法扩展到带工具的 agent 轨迹。
为什么重要：这意味着 eval 正从“题库打分”转向“生产分布下的行为预测”。如果你后面做 MCP、RAG、Browser、Shell agent，这类 replay-based eval 会比单纯 benchmark 更接近真实风险。

2. **OpenAI 在 2026-06-11 宣布将收购 Ona，明确把持久化云执行环境并入 Codex 路线**
摘要：官方表述很直接，目标是给长时间运行的 agent 提供安全、客户可控的云端执行环境，让工作能跨小时甚至跨天继续。
为什么重要：这说明“长时任务 + 持久工作区 + 客户云内执行”已经不是外围配套，而是头部平台的核心产品方向。你转型时要把 runtime、隔离、恢复、审计当成一等能力学。

3. **AWS 在 2026-06-17 公布 AWS Context，试图把企业知识图谱变成 agent 的治理上下文层**
摘要：AWS Context 计划自动映射企业数据关系图谱，并通过 agentic search API 与 MCP tools 在运行时暴露给 agent。
为什么重要：这不是传统 RAG 的“向量召回升级版”，而是“带权限、业务规则、关系语义的上下文层”。对有 IoT 和系统集成背景的人，这类跨系统语义桥接反而是优势项。

4. **Amazon Bedrock AgentCore Harness 在 2026-06-18 GA，把生产级 agent 运行面收束到 CreateHarness / InvokeHarness**
摘要：Harness 现在把隔离环境、文件系统、shell、技能、记忆、模型切换、工具接入和可观测性打包成两次 API 调用。
为什么重要：行业正在把“agent loop 外面的脏活累活”产品化。以后比较有价值的工程能力，不只是写 prompt，而是能否把身份、工具、状态、审计与回放统一起来。

5. **Hugging Face 在 2026-06-17 发布 ARD 草案，试图为 agent 建立通用发现层**
摘要：Agentic Resource Discovery 被定义为位于 agent 与工具前面的开放发现规范，目标是让 agent 在运行时搜索能力，而不是全部预装。
为什么重要：这条路线和 MCP 互补。MCP 解决“怎么调用”，ARD 更像在补“怎么发现”。如果它继续扩散，未来 agent 工程会多一个标准层：registry、catalog、ranking、trust。

6. **Microsoft Agent Framework Python 1.9.0 在 2026-06-18 把循环执行、工具审批和 shell 接入一起前移**
摘要：这个版本加入 `AgentLoopMiddleware`、tool approval middleware、harness agent 内置工具审批，并默认拒绝 server-initiated MCP sampling。
为什么重要：主流框架的竞争点已经不再是“有没有 agent”，而是“默认是否安全、是否可审批、长任务是否可控”。这很适合你用来理解企业 agent 的真正工程边界。

7. **Google ADK 2.3.0 在 2026-06-17 持续补齐企业接入层**
摘要：新版本加入 AgentRegistry client 的 mTLS、CLI `log_level`、GCS first-party toolset，以及更多面向 enterprise 参数的迁移。
为什么重要：Google 这条线在强化“注册发现 + 企业接入 + 远程执行”而不只是 demo agent。你如果想比较框架路线，ADK 值得当作“企业化对照组”。

8. **OpenAI Agents Python 0.17.6 在 2026-06-19 把工具调用护栏前移到审批前**
摘要：新版本加入 pre-approval tool input guardrails，并补了仅 SDK 可见的 tool output custom data。
为什么重要：这是一个很明确的工程信号：危险工具不该只在调用后审计，而该在调用前做输入级拦截。后面你做 shell、数据库写入、工单系统时，这会是默认设计模式。

9. **Hugging Face 在 2026-06-18 发布“Is it agentic enough?”，把模型评测拉回“用你自己的工具栈测试”**
摘要：这篇官方实验不是再刷通用榜单，而是强调在你自己的工具、轨迹和 traces 上比较 open models。
为什么重要：这是对“脱离系统上下文做模型排名”很直接的反驳。你后面要形成的工程习惯应该是：先搭最小任务闭环，再评模型，而不是先追排行榜。

10. **LangGraph 1.2.6 在 2026-06-18 修补子图状态继承和中断取消语义**
摘要：这次修的是 nested subgraph 继承 `checkpoint_ns` 的回归问题，以及 v3 stream abort 时未正确取消运行中 subgraph 的问题。
为什么重要：真正落地长任务 agent，难点经常不是“规划能力”，而是状态一致性、可恢复性和中断语义。这样的修补虽然不炫，但直接决定状态机方案能否稳定上线。

## 重点解读（1条）

### 为什么今天最值得深看的是 OpenAI 收购 Ona

如果只看模型层，这条新闻很像普通并购；但从 Agent 工程视角看，它几乎是在公开承认一件事：**下一阶段的竞争核心，不只是模型更聪明，而是 agent 有没有一个可信、可持续、可审计的工作区。**

OpenAI 在 2026-06-11 的官方说明里把方向说得非常清楚：

- Codex 的高价值工作正在从“几分钟内完成”转向“持续数小时或数天”
- 用户不应该被绑在启动任务的那台机器上
- 企业需要 agent 运行在自己可控的云环境里，而不是黑盒外包执行

这背后的工程含义非常具体：

1. **持久化执行环境会变成标配**
过去很多 agent demo 默认“单轮会话 + 本地进程 + 任务结束即销毁”。但长任务一旦进入真实生产，就一定会遇到会话中断、凭证过期、人工审批插入、外部系统限流、任务恢复这些问题。没有持久工作区，很多能力根本交付不了。

2. **安全边界会从“模型 API”转到“工作区 API”**
以后真正要治理的对象，不只是 prompt 和 output，而是 workspace 里能访问什么文件、什么密钥、什么网络、什么工具、什么审批流。也就是说，runtime 设计本身就是安全设计。

3. **“Agent 平台”会越来越像“受治理的任务操作系统”**
谁能提供可恢复执行、最小权限、轨迹审计、人工接管、跨会话继续、客户云内运行，谁就更接近企业可用。这里面大量问题都不是纯 AI 问题，而是你熟悉的集成、权限、状态机、故障恢复问题。

我的判断是：**2026 下半年，真正值钱的 Agent 工程能力会继续从“会编排几个框架”转向“能设计可信运行面”。** 这对你的转型路线是利好，因为它更重系统工程，不只是模型调参。

## 对当前转型路线的影响

- 学习主线可以继续压在 `Python + MCP + runtime/orchestration + eval + observability`，今天这 10 条基本都在强化这条路线。
- 你的 Java / IoT 集成背景不是绕路，而是护城河。权限边界、设备/系统接入、长链路状态管理，本来就是 Agent 工程里的硬问题。
- 最近如果只能优先补一个短板，我建议补 `可恢复任务执行设计`：会话恢复、工作区状态、工具审批、任务重放。
- 选练手项目时，尽量让项目同时包含这 4 个元素：`工具接入`、`持久状态`、`人工审批`、`轨迹日志`。

## 今晚可验证动作（10-20分钟）

给你手头任意一个小 Agent 补一份最小 `workspace-contract.md`，只写 6 项：

1. 工作区存放位置
2. 任务恢复条件
3. 凭证注入方式
4. 允许使用的工具清单
5. 人工审批触发点
6. 轨迹日志落盘位置

写完后再问自己两个问题：

1. 如果进程中断 2 小时，任务怎么继续？
2. 如果工具误调用，最早在哪一层拦住？

如果这两个问题答不顺，说明你的 agent 还停留在 demo 阶段，而不是工程阶段。

## 原文链接

1. [OpenAI: Predicting model behavior before release by simulating deployment](https://openai.com/index/deployment-simulation/)
2. [OpenAI: OpenAI to acquire Ona](https://openai.com/index/openai-to-acquire-ona/)
3. [AWS: Context intelligence for your data and AI agents at scale](https://aws.amazon.com/blogs/machine-learning/context-intelligence-for-your-data-and-ai-agents-at-scale/)
4. [AWS: Amazon Bedrock AgentCore harness is now generally available](https://aws.amazon.com/blogs/machine-learning/amazon-bedrock-agentcore-harness-is-now-generally-available-go-from-idea-to-production-grade-agent-in-minutes/)
5. [Hugging Face: Agentic Resource Discovery: Let agents search](https://huggingface.co/blog/agentic-resource-discovery-launch)
6. [GitHub: microsoft/agent-framework releases](https://github.com/microsoft/agent-framework/releases)
7. [GitHub: google/adk-python releases](https://github.com/google/adk-python/releases)
8. [GitHub: openai/openai-agents-python releases](https://github.com/openai/openai-agents-python/releases)
9. [Hugging Face: Is it agentic enough? Benchmarking open models on your own tooling](https://huggingface.co/blog/is-it-agentic-enough)
10. [GitHub: langchain-ai/langgraph releases](https://github.com/langchain-ai/langgraph/releases)

## 原文中文翻译链接（机器翻译）

1. [OpenAI Deployment Simulation 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/deployment-simulation/)
2. [OpenAI 收购 Ona 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/openai-to-acquire-ona/)
3. [AWS Context 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://aws.amazon.com/blogs/machine-learning/context-intelligence-for-your-data-and-ai-agents-at-scale/)
4. [AgentCore Harness GA 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://aws.amazon.com/blogs/machine-learning/amazon-bedrock-agentcore-harness-is-now-generally-available-go-from-idea-to-production-grade-agent-in-minutes/)
5. [ARD 启动文章中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://huggingface.co/blog/agentic-resource-discovery-launch)
6. [Microsoft Agent Framework Releases 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/microsoft/agent-framework/releases)
7. [Google ADK Releases 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/google/adk-python/releases)
8. [OpenAI Agents Python Releases 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/openai/openai-agents-python/releases)
9. [Is it agentic enough? 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://huggingface.co/blog/is-it-agentic-enough)
10. [LangGraph Releases 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/langchain-ai/langgraph/releases)
