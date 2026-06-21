# 2026-06-21 Agent 工程雷达：运行时产品化、上线前评估与工具护栏开始合流

## 今日雷达总览（10条）

1. **OpenAI 在 2026-06-16 发布 Deployment Simulation，把“上线前评估”推进到真实对话回放层**
摘要：OpenAI 公开了一套在模型发布前重放历史真实对话上下文的方法，用来估计新模型在生产里的异常行为频率，并且已经扩展到带工具的 agent 轨迹。
为什么重要：这比“手工出几道 benchmark 题”更接近真实上线条件。对 Agent 工程来说，评估重点开始从静态题库转向“真实任务轨迹 + 工具环境仿真”。

2. **Anthropic 在 2026-06-09 发布 Claude Fable 5 / Mythos 5，但 2026-06-12 又暂停外部访问**
摘要：Anthropic 把 Fable 5 定位为已做安全收敛的 Mythos 级模型，强调长任务自治、软件工程和知识工作能力；但三天后官方又宣布暂停访问。
为什么重要：前沿模型的工程现实已经不是“能不能用”，而是“能用多久、在什么边界下能用、出了政策或安全变动怎么回退”。这直接关系到供应商切换、回退策略和能力探测。

3. **Amazon Bedrock AgentCore Harness 在 2026-06-18 GA，Agent 运行时开始被打包成两次 API 调用**
摘要：AWS 把生产级 agent 所需的运行时、内存、身份、浏览器、MCP/Gateway 接入、观测等能力收敛进 Harness，主打 `CreateHarness` / `InvokeHarness` 两个入口。
为什么重要：这说明行业开始把“agent loop 之外的脏活累活”产品化。你以后做 Agent，不会只比拼 prompt 和 orchestration，还要理解运行时隔离、内存、工具接入、身份和观测如何一起交付。

4. **Amazon Bedrock AgentCore Web Search 在 2026-06-19 GA，并且明确做成 MCP 兼容能力**
摘要：AWS 发布托管式 Web Search，支持标准 `tools/list` 发现，强调索引由 AWS 自管、分钟级更新、查询不离开 AWS。
为什么重要：Web grounding 正在从“自己拼搜索 API”变成平台原生能力。对企业 Agent 来说，这比搜索效果本身更关键，因为它把新鲜度、权限、合规和调用接口统一了。

5. **Microsoft Agent Framework Python 1.9.0 在 2026-06-18 把 MCP 护栏与 Harness 审批继续前移**
摘要：新版本加入 `AgentLoopMiddleware`、工具审批中间件、shell tool 接入，并把 MCP server-initiated sampling 默认改成拒绝，还把 orchestration 提升为 stable。
为什么重要：这代表主流框架的竞争点正在转到“默认是否安全、审批是否内建、长任务循环是否可控”，很适合你这种有集成和生产稳定性视角的人重点跟。

6. **OpenAI Agents Python v0.17.6 在 2026-06-19 加入 pre-approval tool input guardrails**
摘要：OpenAI 官方 Agents SDK 新版新增“工具调用前输入护栏”，并补了 tool output 的 SDK-only custom data 能力。
为什么重要：工具调用安全正在从“调用后审计”前移到“调用前拦截”。如果你后面做高权限工具、Shell、数据库写入、工单操作，这类 guardrail 会成为默认需求。

7. **Google ADK v2.3.0 在 2026-06-17 继续往企业接入层推进**
摘要：ADK 新版加入 AgentRegistry mTLS、MCP toolset 到授权会话迁移、按请求 OpenTelemetry 配置、远程沙箱环境和更多 eval/optimizer 能力。
为什么重要：Google 这条线的信号很明确，ADK 不只是做 demo agent，而是在补企业网络、遥测和远程执行这些“生产必需件”。

8. **AWS 在 2026-06-17 预告 AWS Context，把组织级知识图谱变成 Agent 的治理上下文层**
摘要：AWS Context 计划把企业现有数据关系自动映射进知识图谱，并通过 agentic search 与 MCP tools 暴露给 agent，同时继承 IAM / Lake Formation 权限。
为什么重要：这不是普通 RAG。它更像“带权限和业务规则的组织级上下文层”。对你来说，这类能力和 IoT/企业集成经验高度同构，值得重点理解。

9. **LangGraph 1.2.6 在 2026-06-18 修补子图状态继承与中断取消语义**
摘要：新版修复 nested subgraph 继承父 `checkpoint_ns` 的回归问题，也修复 v3 stream abort 时未正确取消运行中 subgraph 的问题。
为什么重要：长任务 Agent 真正难的是可恢复、可中断、可重放。LangGraph 这类修补虽然不“炫”，但直接决定状态机方案能不能稳定落地。

10. **PydanticAI 在 2026-06-10 延续 V2 Beta 7，并把能力层进一步外置到 Harness 生态**
摘要：PydanticAI V2 Beta 7 没再引入新的 V2 破坏性改动，但补进了 UI adapter 相关安全修复；与此同时，官方单独维护 `pydantic-ai-harness`，把 coding、research 等能力模块从 core 里拆出来。
为什么重要：这条路线很适合工程化团队参考。核心框架保持薄，能力通过可组合 capability / harness 外置，后面做企业内 Agent 平台时会更容易治理版本、权限和实验能力。

## 重点解读（1条）

### 为什么我今天最建议你深看 OpenAI 的 Deployment Simulation

过去很多团队说“我们做了 eval”，实际做法通常是：

- 准备一批提示词
- 让模型跑一遍
- 看分数、看失败样例

这在 chat assistant 时代还勉强够用，但到 Agent 时代明显不够，因为真正的风险已经转移到三件事：

- 用户真实上下文是什么
- 模型在长轨迹里怎么逐步做决定
- 工具环境会不会放大偏差

OpenAI 这次公开的方法，核心不是再发一个新 benchmark，而是把“上线前评估”往真实部署条件推近了一步：拿历史真实对话做隐私保护回放，让候选模型在近似生产的上下文里重跑，再去估计异常行为出现频率。更关键的是，它已经明确讨论了 **tool simulation for agentic trajectories**，也就是带工具的 Agent 轨迹同样能做高保真模拟。

这对你的转型价值非常直接：

- 你不该只学“怎么接模型”，还要学“怎么验证一个 Agent 在真实任务流里会不会出事”
- 你不该只做离线题库评测，还要学会保存真实任务轨迹、工具输入输出、审批节点和最终结果
- 你以后做 Shell / 浏览器 / MCP / 数据库工具时，最值钱的能力之一就是搭一个可回放的评估 harness

一句话判断：**2026 下半年的 Agent eval，会越来越像“回放真实生产轨迹的系统工程”，而不是“模型答题比赛”。**

## 对当前转型路线的影响

- 学习主线可以继续压在 `Python + MCP + runtime/orchestration + eval + observability`，今天这 10 条几乎都在强化这条路线。
- 你的集成背景是优势，不是包袱。行业正在把重点从“写个会调模型的 demo”转向“接权限、接工具、接上下文、接观测、接回退”。
- 如果最近只能补一个短板，我建议优先补 `评估轨迹设计`，不是再刷提示词技巧。尤其是任务回放、审批前护栏、工具调用审计这三块。
- 选练手项目时，尽量避免纯聊天助手，最好带上这 4 个元素：`MCP 工具接入`、`长任务状态`、`人工审批`、`可回放评估日志`。

## 今晚可验证动作（10-20分钟）

给你现有任意一个小 Agent 补一份最小“任务轨迹日志”设计，先不用写全系统，只要写出这 6 个字段：

1. `task_id`
2. `user_goal`
3. `tool_calls`
4. `approval_events`
5. `final_result`
6. `verdict`

然后拿最近一次真实或模拟任务，手工补一条 JSON 记录，再回答 3 个问题：

1. 如果换一个模型重跑，这条轨迹哪些字段必须复用？
2. 如果某一步工具调用危险，最早应该在哪个节点拦？
3. 如果要做 nightly replay eval，哪一步最难稳定复现？

这 15 分钟会把你从“会用框架”拉向“会设计 Agent 工程闭环”。

## 原文链接

1. [OpenAI: Predicting model behavior before release by simulating deployment](https://openai.com/index/deployment-simulation/)
2. [Anthropic: Claude Fable 5 and Claude Mythos 5](https://www.anthropic.com/news/claude-fable-5-mythos-5)
3. [Amazon AWS Blog: Amazon Bedrock AgentCore harness is now generally available](https://aws.amazon.com/blogs/machine-learning/amazon-bedrock-agentcore-harness-is-now-generally-available-go-from-idea-to-production-grade-agent-in-minutes/)
4. [Amazon AWS Blog: Introducing Web Search on Amazon Bedrock AgentCore](https://aws.amazon.com/blogs/machine-learning/introducing-web-search-on-amazon-bedrock-agentcore/)
5. [GitHub: microsoft/agent-framework releases](https://github.com/microsoft/agent-framework/releases)
6. [GitHub: openai/openai-agents-python releases](https://github.com/openai/openai-agents-python/releases)
7. [GitHub: google/adk-python releases](https://github.com/google/adk-python/releases)
8. [Amazon AWS Blog: Context intelligence for your data and AI agents at scale](https://aws.amazon.com/blogs/machine-learning/context-intelligence-for-your-data-and-ai-agents-at-scale/)
9. [GitHub: langchain-ai/langgraph releases](https://github.com/langchain-ai/langgraph/releases)
10. [GitHub: pydantic/pydantic-ai releases](https://github.com/pydantic/pydantic-ai/releases)
11. [GitHub: pydantic/pydantic-ai-harness](https://github.com/pydantic/pydantic-ai-harness)

## 原文中文翻译链接（机器翻译）

1. [OpenAI Deployment Simulation 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/deployment-simulation/)
2. [Claude Fable 5 / Mythos 5 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://www.anthropic.com/news/claude-fable-5-mythos-5)
3. [AgentCore Harness GA 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://aws.amazon.com/blogs/machine-learning/amazon-bedrock-agentcore-harness-is-now-generally-available-go-from-idea-to-production-grade-agent-in-minutes/)
4. [AgentCore Web Search 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://aws.amazon.com/blogs/machine-learning/introducing-web-search-on-amazon-bedrock-agentcore/)
5. [Microsoft Agent Framework releases 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/microsoft/agent-framework/releases)
6. [OpenAI Agents Python releases 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/openai/openai-agents-python/releases)
7. [Google ADK releases 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/google/adk-python/releases)
8. [AWS Context 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://aws.amazon.com/blogs/machine-learning/context-intelligence-for-your-data-and-ai-agents-at-scale/)
9. [LangGraph releases 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/langchain-ai/langgraph/releases)
10. [PydanticAI releases 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/pydantic/pydantic-ai/releases)
11. [Pydantic AI Harness 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/pydantic/pydantic-ai-harness)
