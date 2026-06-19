# 2026-06-19 Agent 工程雷达：Eval 前置、运行时抬升与框架分层

## 今日雷达总览（10条）

1. **Anthropic 于 2026-06-09 发布 Claude Fable 5 与 Claude Mythos 5**
摘要：Anthropic 把更强前沿能力拆成可广泛使用的 Fable 5，与更受限的 Mythos 5。官方强调它们在软件工程、知识工作、视觉、长任务和持久记忆上都有明显提升。  
为什么重要：这不是单纯“模型更强”，而是“能力层 + 访问层 + 护栏层”一起发布。以后做 Agent 选型，不能只看 benchmark，还得看你能否稳定拿到、在哪些权限边界内能拿到。

2. **Anthropic 于 2026-06-12 因出口管制暂停 Fable 5 / Mythos 5 访问**
摘要：Anthropic 官方公告称，美国政府基于国家安全相关出口管制，要求暂停 Fable 5 和 Mythos 5 对所有客户的访问，其他模型不受影响。  
为什么重要：这再次说明前沿模型的“可用性”会被合规和政策瞬时改写。生产 Agent 必须有模型回退、任务降级和能力探测机制，不能把单模型写死成系统前提。

3. **OpenAI 于 2026-06-16 发布 Deployment Simulation**
摘要：OpenAI 公开了一种上线前模拟真实部署的方法：用历史真实对话前缀回放候选模型，在更接近生产流量的上下文里预测不良行为频率，也已扩展到带工具调用的 agent 场景。  
为什么重要：这代表 eval 正在从“离线题库”升级成“部署前预演”。对你转型最关键的启发是：真正值钱的评测，不是题刷得多，而是能不能接近真实工作流、真实失败路径和真实上下文。

4. **Vercel 于 2026-06-17 发布开源 Agent 框架 eve 公测版**
摘要：eve 把 Agent 定义成一个目录结构，内建 durable execution、sandbox、人工审批、subagents 和 evals，目标是让“本地可跑的 Agent”直接具备生产特征。  
为什么重要：这是很强的工程路线信号。Agent 框架竞争点正从 prompt 封装，转向运行时、文件系统约定、持久化执行和治理能力。

5. **GitHub 于 2026-06-11 将 Agentic Workflows 推入 public preview**
摘要：GitHub 现在允许把 issue triage、CI 故障分析、文档更新等推理型任务直接写成 agentic workflow，并运行在 GitHub Actions 内。  
为什么重要：Agent 已经进入“仓库事件驱动自动化层”，不再只是聊天入口。对工程落地来说，这比单独的 IDE 助手更接近业务价值闭环。

6. **GitHub 于 2026-06-02 宣布 Copilot SDK GA**
摘要：GitHub Copilot SDK 进入 GA，提供稳定 API，让你直接复用 Copilot 背后的 agent runtime，包括 planning、tool invocation、file edits、streaming、多轮 session 和 OpenTelemetry tracing。  
为什么重要：这意味着“现成 agent runtime”开始商品化。对你来说，重点不必放在从零造 orchestration，而要学会把现成 runtime 嵌入到现有工程、平台和内部工具里。

7. **Vercel 于 2026-06-16 将 Sandbox 最长运行时长提升到 24 小时**
摘要：Vercel Sandbox 可不中断运行 24 小时，较此前 5 小时大幅提升，覆盖长数据处理、端到端测试和长生命周期 Agent 工作流。  
为什么重要：长任务 Agent 不一定要先上复杂 worker 集群了。很多验证型项目现在可以先靠更简单的 sandbox runtime 起步，再视负载升级架构。

8. **Vercel 于 2026-06-16 为 Workflow SDK 5 beta 加入 inflight cancellation**
摘要：Workflow 与 step 边界之间现在可以共享 `AbortController` / `AbortSignal`，支持跨步骤取消进行中的调用。  
为什么重要：这是 Agent 编排从“能跑”走向“能及时停”的关键细节。取消、超时、竞速返回、浪费控制，这些都会直接影响成本和稳定性。

9. **OpenAI 重申 Apps SDK 基于 MCP，且已开源**
摘要：OpenAI 在 Apps SDK 页面中明确说明，Apps SDK 建立在 MCP 之上，并把 SDK 开源，使应用既能定义逻辑，也能定义界面。  
为什么重要：MCP 正在从“工具协议”往“应用接口层”延伸。对你继续投入 MCP 是利好，因为这类标准一旦进入 UI + tool 双层，迁移价值会明显提高。

10. **LangGraph 于 2026-06-18 发布 1.2.6，继续修补子图与中断行为**
摘要：LangGraph 1.2.6 修复了 nested subgraph 继承父 checkpoint namespace 的回归问题，也修复了 v3 stream abort 时取消运行中 subgraph 的行为。  
为什么重要：这类更新不炫，但很生产。只要你准备做多步 Agent、持久状态和可恢复执行，checkpoint 与中断语义是否稳定，往往比 demo 表现更重要。

## 重点解读（1条）

### Deployment Simulation 说明：Agent eval 的主战场正在前移到“上线前仿真”

OpenAI 这次最值得深看的，不是又发了一个新 benchmark，而是它把“上线前如何知道模型在真实环境里会不会出事”做成了一套更像工程系统的办法。

传统 eval 的问题很明显：

- 题目通常是人工设计的，和真实用户上下文有差距
- 模型知道自己在被测，容易出现“会考试但不会上岗”
- 很难覆盖多轮、工具调用、长任务和灰度发布前的真实失败模式

Deployment Simulation 的思路更接近真实生产：

- 复用历史真实对话前缀
- 用候选模型回放这些上下文
- 统计不良行为频率，而不只看单题对错
- 把方法扩展到 agentic rollout 和工具使用场景

这对你的路线有三个直接启发：

1. **Eval 要嵌进工作流，不要停留在题库**
如果你做 RAG、MCP 或工作流 Agent，真正该评的是“这个任务链跑完是否稳定、是否乱调工具、是否在失败时正确中断”，而不是只评最终回答像不像。

2. **观测字段要先设计，不然没法仿真**
如果你没有记录 `task_type`、`selected_model`、`tool_calls`、`failure_reason`、`human_handoff` 这类字段，后面就很难做回放式评估。

3. **上线前的风险验证会越来越像 CI**
未来成熟团队很可能不是“改完 prompt 直接发”，而是“先拿真实历史轨迹回放，跑一轮 deployment-like simulation，再决定是否灰度”。

一句话总结：**Agent eval 正在从研究活动，变成发布流程的一部分。**

## 对当前转型路线的影响

- 你的主线可以继续压在 `Python + MCP + workflow/runtime + eval + observability`，这条线和今天的信号高度一致。
- 学习重点建议从“怎么让 Agent 会调工具”升级为“怎么让 Agent 在长链路里可测、可停、可回退、可追踪”。
- 你的 Java / IoT 集成背景在这里很有优势，因为长任务编排、超时取消、回退、状态恢复，本质上就是成熟集成系统问题。
- 近期练手项目最好带上这 4 个元素：`多模型回退`、`超时/取消`、`执行轨迹记录`、`人工确认点`。

## 今晚可验证动作（10-20分钟）

给你当前任意一个 Agent demo 补一个最小“仿真与回放日志”结构，先只加 6 个字段：

1. `task_type`
2. `selected_model`
3. `tool_calls`
4. `run_status`
5. `failure_reason`
6. `handoff_required`

然后挑 3 次历史运行，手工复盘：

- 哪一步最容易失败
- 失败时是否能被明确分类
- 是否存在不该继续执行却继续跑的情况

如果 10 分钟还有余量，再补一个最小脚本，把失败样例整理成一个 `jsonl` 文件，作为你后面做 deployment-like replay 的种子数据。

## 原文链接

1. [Anthropic: Claude Fable 5 and Claude Mythos 5](https://www.anthropic.com/news/claude-fable-5-mythos-5)
2. [Anthropic: Statement on the US government directive to suspend access to Fable 5 and Mythos 5](https://www.anthropic.com/news/fable-mythos-access)
3. [OpenAI: Predicting model behavior before release by simulating deployment](https://openai.com/index/deployment-simulation/)
4. [Vercel: Introducing eve, an open-source agent framework](https://vercel.com/changelog/introducing-eve-an-open-source-agent-framework)
5. [GitHub Changelog: GitHub Agentic Workflows is now in public preview](https://github.blog/changelog/2026-06-11-github-agentic-workflows-is-now-in-public-preview/)
6. [GitHub Changelog: Copilot SDK is now generally available](https://github.blog/changelog/2026-06-02-copilot-sdk-is-now-generally-available/)
7. [Vercel: Vercel Sandbox can now run for up to 24 hours](https://vercel.com/changelog/vercel-sandbox-can-now-run-for-up-to-24-hours)
8. [Vercel: Workflow SDK now supports inflight cancellation](https://vercel.com/changelog/workflow-sdk-now-supports-inflight-cancellation)
9. [OpenAI: Introducing apps in ChatGPT and the new Apps SDK](https://openai.com/index/introducing-apps-in-chatgpt/)
10. [GitHub: langchain-ai/langgraph releases](https://github.com/langchain-ai/langgraph/releases)

## 原文中文翻译链接（机器翻译）

1. [Claude Fable 5 / Mythos 5 发布中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://www.anthropic.com/news/claude-fable-5-mythos-5)
2. [Fable 5 / Mythos 5 暂停访问中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://www.anthropic.com/news/fable-mythos-access)
3. [Deployment Simulation 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/deployment-simulation/)
4. [eve 开源 Agent 框架中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://vercel.com/changelog/introducing-eve-an-open-source-agent-framework)
5. [GitHub Agentic Workflows 公测中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.blog/changelog/2026-06-11-github-agentic-workflows-is-now-in-public-preview/)
6. [Copilot SDK GA 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.blog/changelog/2026-06-02-copilot-sdk-is-now-generally-available/)
7. [Vercel Sandbox 24 小时中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://vercel.com/changelog/vercel-sandbox-can-now-run-for-up-to-24-hours)
8. [Workflow inflight cancellation 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://vercel.com/changelog/workflow-sdk-now-supports-inflight-cancellation)
9. [OpenAI Apps SDK 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/introducing-apps-in-chatgpt/)
10. [LangGraph releases 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/langchain-ai/langgraph/releases)
