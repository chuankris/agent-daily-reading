# 今日雷达主题

这两天最值得你注意的，不只是新模型继续刷新上限，而是 Agent 工程的三条主线开始更清楚地收拢到一起：`模型分层路由`、`预部署评测`、`运行期治理/观测`。这正好对应你从 Java/IoT 集成转到 Agent 工程时最该补齐的工程骨架。

## 今日雷达总览（10条）

### 1. OpenAI 在 2026-07-09 发布 GPT-5.6，明确把模型分层和子代理能力产品化
- 摘要：OpenAI 发布 `GPT-5.6 Sol / Terra / Luna`，并在更高档位引入更强推理强度与并行子代理能力，面向长链路编码、科研和安全任务。
- 为什么重要：以后做 Agent，不该再把“所有请求都打到同一个大模型”当默认方案。模型档位、推理强度、子任务拆分，会越来越像后端系统里的分层服务与流量路由。

### 2. Anthropic 在 2026-06-30 发布 Claude Sonnet 5，继续巩固“默认执行模型”档位
- 摘要：Claude Sonnet 5 被定位为更 agentic 的 Sonnet，可做规划、工具调用、浏览器和终端协作，目标是把原本要更大模型才能完成的工作压到更便宜的中坚档位。
- 为什么重要：真实生产里，最有价值的往往不是“最强旗舰”，而是“够强、够稳、成本可控”的主力模型。Sonnet 5 对你理解模型选型与成本控制很有参考价值。

### 3. OpenAI 在 2026-06-16 公布 Deployment Simulation，把“上线前模拟真实部署”推成评测方法
- 摘要：OpenAI 用历史真实对话做隐私保护回放，在模型正式发布前先估计不良行为概率，并把方法扩展到了带工具调用的 agent 轨迹。
- 为什么重要：这不是又一个 benchmark，而是评测思路升级。以后评测 Agent，不能只看离线题库得分，更要看“在真实流量形态下会不会偏航、误调工具、暴露新风险”。

### 4. Anthropic 推出 Claude Opus 4.8，并同步把 Dynamic Workflows 带入 Claude Code
- 摘要：Opus 4.8 强调更稳定的工具使用与更低的失配率；同时 Dynamic Workflows 允许在单次会话中规划并运行大批并行子代理，再统一校验结果。
- 为什么重要：这说明“多代理并行 + 结果校验”正在从框架层特性下沉成模型/产品默认能力。你之后设计日志、预算和回滚策略时，要默认系统内部可能已经在并行展开。

### 5. Google 在 2026-07-01 解释 ADK 2.0 的设计方向：把确定性工作流重新抬回一线
- 摘要：ADK 2.0 认为很多系统把路由、调度、重试、错误处理都压给 LLM，本质上既慢又贵且不稳定，因此新增结构化工作流运行时和任务协作模型。
- 为什么重要：这正是你最值得跟进的技术路线。它和传统集成工程很接近：把不确定性交给模型，把确定性控制留给代码和流程。

### 6. Google 推动 Gemini CLI 向 Antigravity CLI 迁移，CLI Agent 开始和桌面 Agent 统一底层 harness
- 摘要：Google 宣布 Antigravity CLI 面向所有人开放，强调它与 Antigravity 2.0 桌面版共享同一 agent harness，并支持后台多代理异步工作流。
- 为什么重要：CLI、IDE、桌面三端共用同一执行底座，是 Agent 产品化的重要信号。未来学习重点不只是“某个入口怎么用”，而是“底层 harness 怎样编排、恢复和治理”。

### 7. GitHub 在 2026-06-17 上线 Agent Finder，让 Copilot 自动发现合适的 MCP/skills/tools
- 摘要：GitHub Copilot 现在可以自动发现任务所需的 MCP servers、skills、agents 和 tools，而不是完全依赖手工装配上下文。
- 为什么重要：能力发现层正在变成一等公民。对企业内部 Agent 平台来说，目录、发现、权限映射会越来越像过去的服务注册中心。

### 8. GitHub 在 2026-07-08 上线 Enterprise-managed OpenTelemetry export，把 Agent 观测正式纳入平台治理
- 摘要：企业现在可以集中下发 Copilot VS Code 与 CLI 的 OTel 导出目标、鉴权头、资源属性，以及是否允许采集 prompt/response/tool 内容。
- 为什么重要：这说明 Agent 观测不再是“开发者本地自己配一下”。谁能看见什么、哪些内容能被采集、凭证怎样下发，都会变成平台治理能力。

### 9. GitHub 在 2026-07-09 把 GPT-5.6 带进 Copilot，全渠道开始支持按模型档位工作
- 摘要：Copilot 开始逐步提供 GPT-5.6 Sol / Terra / Luna，覆盖 VS Code、Visual Studio、CLI、cloud agent、github.com、JetBrains 等入口。
- 为什么重要：模型分层不再只存在于模型厂商 API 文档里，而是直接进入开发者日常工具链。你要开始把“任务按档位分配模型”视作默认工程动作。

### 10. Cloudflare 正把 Agent 运行时平台化：Project Think 经验下沉到 Agents SDK，并向 Flue 这类框架开放
- 摘要：Cloudflare 先用 Project Think 验证长任务 Agent 所需的 durable execution、sub-agents、sandboxed code execution、persistent sessions，再把这些能力下沉到 Agents SDK，供 Flue 等框架复用。
- 为什么重要：生产级 Agent 正在形成三层栈：`framework -> harness -> runtime/platform`。这和你熟悉的集成平台思路高度一致，比只学 prompt 技巧更有长期复用价值。

## 重点解读（1条）

### OpenAI Deployment Simulation：为什么它比“再多一个评测榜单”更值得你深挖

这条最值得你今天认真看。因为它讨论的不是“模型答题又涨了几分”，而是一个更接近生产工程的问题：**模型上线前，能不能先在近似真实流量里跑一遍，提前看见它会怎样出错。**

OpenAI 这套方法的核心，是把历史真实对话在隐私保护前提下回放给候选模型，再估计不良行为出现频率。更关键的是，它不只面向普通聊天，也扩展到了带工具调用的 agent 轨迹。这一点和你的转型方向强相关，因为真正麻烦的问题往往不出在“回答一句话”，而出在“连续几步调用工具后是否偏航、误触发、越权或忘记回收状态”。

对工程实现的启发很直接：

1. 你的评测集不该只是一堆问答样本，还要保留真实上下文、工具状态、历史消息和失败分支。
2. 你的上线流程不该只有“线下评测通过”，还应有一层“预部署回放”，专门观察异常路径和长链路任务。
3. 你的日志设计不该只记录最终答案，还要能复盘中间工具调用、分支决策、权限变化和重试行为。

如果说 ADK 2.0 代表“把确定性控制从模型手里拿回来”，那 Deployment Simulation 代表“把评测从实验室拉回真实世界”。这两条路线放在一起看，基本就是 2026 年下半年 Agent 工程的主线。

## 对当前转型路线的影响

- 你的学习重心可以进一步收敛到 4 块：`模型路由`、`工作流/状态机`、`评测回放`、`观测与权限治理`。
- 你过去做 Java/IoT 集成的经验不是包袱，反而是优势，因为 Agent 系统正在重新强调编排、重试、回滚、审计和边界控制。
- 如果接下来只做一个练手项目，优先做那种同时包含 `工具调用 + 状态持久化 + 评测回放 + 日志观测` 的小系统，不要再停留在单轮聊天 demo。

## 今晚可验证动作（10-20分钟）

1. 读一遍 OpenAI 的 Deployment Simulation 文章，只回答一个问题：你现在最熟悉的业务流程里，哪 2 个步骤最适合先做“历史流量回放评测”。
2. 用你熟悉的 Java 或 Python 写一个最小事件日志结构，至少包含：`task_id`、`step`、`tool_name`、`input_digest`、`result_digest`、`latency_ms`、`retry_count`。
3. 对照 ADK 2.0 的思路，写下一个判断规则：哪些步骤必须由代码硬控，哪些步骤允许模型自由发挥。

## 原文链接

1. [OpenAI: GPT-5.6](https://openai.com/index/gpt-5-6/)
2. [Anthropic: Introducing Claude Sonnet 5](https://www.anthropic.com/news/claude-sonnet-5)
3. [OpenAI: Predicting model behavior before release by simulating deployment](https://openai.com/index/deployment-simulation/)
4. [Anthropic: Introducing Claude Opus 4.8](https://www.anthropic.com/news/claude-opus-4-8)
5. [Google Developers Blog: Why we built ADK 2.0](https://developers.googleblog.com/why-we-built-adk-20/)
6. [Google Developers Blog: Transitioning Gemini CLI to Antigravity CLI](https://developers.googleblog.com/an-important-update-transitioning-gemini-cli-to-antigravity-cli/)
7. [GitHub Changelog: Agent finder for GitHub Copilot now available](https://github.blog/changelog/2026-06-17-agent-finder-for-github-copilot-now-available/)
8. [GitHub Changelog: Enterprise-managed OpenTelemetry export for VS Code and CLI](https://github.blog/changelog/2026-07-08-enterprise-managed-opentelemetry-export-for-vs-code-and-cli/)
9. [GitHub Changelog: OpenAI's GPT-5.6 Sol, Terra, and Luna are now available in GitHub Copilot](https://github.blog/changelog/2026-07-09-openais-gpt-5-6-sol-terra-and-luna-are-now-available-in-github-copilot/)
10. [Cloudflare Blog: Bringing more agent harnesses and frameworks to Cloudflare, starting with Flue](https://blog.cloudflare.com/agents-platform-flue-sdk/)

## 原文中文翻译链接（机器翻译）

1. [GPT-5.6 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/gpt-5-6/)
2. [Claude Sonnet 5 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://www.anthropic.com/news/claude-sonnet-5)
3. [Deployment Simulation 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/deployment-simulation/)
4. [Claude Opus 4.8 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://www.anthropic.com/news/claude-opus-4-8)
5. [ADK 2.0 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://developers.googleblog.com/why-we-built-adk-20/)
6. [Antigravity CLI 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://developers.googleblog.com/an-important-update-transitioning-gemini-cli-to-antigravity-cli/)
7. [Agent Finder 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.blog/changelog/2026-06-17-agent-finder-for-github-copilot-now-available/)
8. [OTel Export 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.blog/changelog/2026-07-08-enterprise-managed-opentelemetry-export-for-vs-code-and-cli/)
9. [Copilot GPT-5.6 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.blog/changelog/2026-07-09-openais-gpt-5-6-sol-terra-and-luna-are-now-available-in-github-copilot/)
10. [Cloudflare Flue/Agents SDK 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://blog.cloudflare.com/agents-platform-flue-sdk/)
