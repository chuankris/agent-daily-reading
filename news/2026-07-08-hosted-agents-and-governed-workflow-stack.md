# 今日雷达主题

托管 Agent、确定性工作流、工具治理，正在从“分散能力点”收敛成一条更完整的工程栈。对你这种从 Java/IoT 集成转向 Agent 工程的人来说，今天最该盯的已经不只是哪个模型又涨了多少分，而是三件事怎么接起来：`更强模型`、`更稳编排`、`更可控的工具/权限/访问层`。

## 今日雷达总览（10条）

### 1. OpenAI 预览 GPT-5.6 系列，模型分层开始更明确
- 事件摘要：OpenAI 在 2026-06-26 预览 GPT-5.6 系列，分成 `Sol / Terra / Luna` 三档，分别对应旗舰、均衡、低成本路线。
- 为什么值得关注：这说明主流供应商正在把“模型路由”产品化。以后做 Agent，不会只绑定一个默认模型，而是默认要设计高低配切换、任务分层和成本控制。

### 2. Claude Sonnet 5 上线，Sonnet 档位继续压实“默认执行模型”角色
- 事件摘要：Anthropic 在 2026-06-30 发布 Claude Sonnet 5，并把它铺到 Claude、Claude Code 和 Claude Platform。
- 为什么值得关注：这不是单纯提分，而是在强化一个现实判断: 生产环境里，很多 Agent 默认跑的不是最贵模型，而是“够强、够稳、成本可控”的中高档模型。

### 3. OpenAI Deployment Simulation 把 Agent 风险评估前移到上线前
- 事件摘要：OpenAI 在 2026-06-16 公开 Deployment Simulation，用真实历史对话上下文回放候选模型，提前估计不良行为和对齐风险。
- 为什么值得关注：这直接对应你要补的 `Eval + Observability` 能力。真正成熟的 Agent 团队，会把“上线前仿真”当成和线上 trace 同等级的基础设施。

### 4. Google ADK 2.0 公开押注“确定性工作流 + Agent 混编”
- 事件摘要：Google 在 2026-07-01 解释 ADK 2.0 的设计动机，明确提出用 deterministic workflow 包住必须严格执行的业务链路，把 LLM 只放在确实需要推理的节点。
- 为什么值得关注：这是非常重要的路线信号。很多企业 Agent 不该用纯 autonomous loop 硬跑，而该回到你熟悉的“编排、边界、回滚、可预测失败”。

### 5. Google 把“Agent 质量飞轮”产品化为 coding agent 技能
- 事件摘要：Google 在 2026-06-30 介绍一个 developer skill，让 coding agent 自动跑“准备数据、推理、自动评分、失败聚类、针对性优化”的五段闭环。
- 为什么值得关注：行业在从“调 prompt 看感觉”转向“质量工程化”。这条线和你接下来要学的 eval 集、自动打分、失败归因高度一致。

### 6. GitHub Copilot code review 开始引入 skills/MCP，上下文不再只靠 diff
- 事件摘要：GitHub 在 2026-06-02 宣布 Copilot code review 支持 agent skills、MCP，以及更高推理深度的 medium analysis tier。
- 为什么值得关注：这说明代码评审 Agent 正在从“看改动”升级为“读组织知识、读外部系统、按复杂度分配算力”。这和企业级 Agent 的真实落地方式更接近。

### 7. GitHub 把 Kimi K2.7 Code 接入 Copilot，开权重模型开始进入主流开发面板
- 事件摘要：GitHub 在 2026-07-01 宣布 Kimi K2.7 Code 正式进入 Copilot model picker，这是 Copilot 首个 open-weight 可选代码模型。
- 为什么值得关注：这释放出一个明确信号: 模型层开始可替换。以后做 Agent 平台，provider abstraction、评测基线、成本路由都会从“可选优化”变成“默认设计”。

### 8. Vercel AI SDK 7 把 Agent 应用常见 plumbing 继续抽象掉
- 事件摘要：Vercel 在 2026-06-25 发布 AI SDK 7，补上 reasoning control、tool/runtime context、provider files、skills support、MCP Apps、terminal UI。
- 为什么值得关注：这类 SDK 正在把 Agent 应用里最重复的胶水层抽成稳定接口。即使你不主做前端，也值得理解这层 abstraction 如何统一 provider、工具调用和会话状态。

### 9. Cloudflare 开始把 Agent 身份、访问控制和付费入口做成边缘层能力
- 事件摘要：Cloudflare 在 2026-07-01 同时推进两件事：一是把 AI 流量细分为 `Search / Agent / Training` 三类进行管理；二是开放 Monetization Gateway，可对网页、数据集、API 甚至 MCP 工具收费。
- 为什么值得关注：未来 Agent 接外部系统，不只是“能不能调用”，而是“以什么身份调用、是否被允许、是否需要付费”。访问治理层正在独立成栈。

### 10. Hugging Face 推动 OpenEnv 社区化，开源生态继续补“可复现执行环境”短板
- 事件摘要：Hugging Face 在 2026-06-08 宣布 OpenEnv 进一步开放，并由多方委员会共同协调，目标是把 terminal、browser 等 agentic execution environment 做成更标准的基础层。
- 为什么值得关注：如果模型是大脑、工具是手，OpenEnv 这类项目补的是“标准化操作环境”。以后你做 eval、训练、长期运行 Agent，越来越需要这种可重复、可隔离的环境层。

## 重点解读（1条）

### Google ADK 2.0：为什么“确定性工作流包住 Agent”这条路线值得你重点跟

这条最值得你深读，因为它直接把 Agent 工程从“全交给模型自己想”拉回到更接近企业软件的方法论。

Google 的核心判断很清楚：如果业务流程本来就要求 `A -> B -> C`，那就不该每一步都让模型重新理解上下文、决定下一跳。ADK 2.0 的做法，是把流程图本身变成运行时边界：确定步骤用 workflow 固定住，只有处理非结构化输入、分类、草拟文案这种环节，才交给 LLM 节点。

这对你有三层直接价值：

1. 它和你过去做 Java/IoT 集成的经验高度同构。你原来擅长的就是编排、状态流转、异常恢复、接口边界，这些在 Agent 时代没有过时，只是换了执行单元。
2. 它能显著降低“上下文膨胀 + agent 跑偏”的风险。Google 直接把问题点讲透了：如果所有工具输出都塞回上下文，长任务迟早会漂。Workflow 的价值就是把执行控制从语言模型里拿出来。
3. 它给了一个非常实用的架构判断标准：凡是链路固定、合规要求强、失败状态必须可预测的地方，优先 workflow；凡是语义理解、不确定分类、自然语言生成，才交给 agent。

我的判断是：接下来 1 到 3 个月，你最该练的不是“多 agent 花式编排”，而是把一个真实业务任务拆成：

`确定性步骤 + 一个或两个需要推理的节点 + 明确的输入输出边界 + 可回放评测`

如果你把这条打通，转型速度会比继续追逐单纯模型榜单更快。

## 对当前转型路线的影响

- 学习主线建议进一步收敛到五块：`模型路由`、`工作流/Agent 混编`、`MCP/工具治理`、`预部署评测`、`可观测与回放`。
- 你过去的系统集成背景是优势，不是包袱。企业真正缺的是“能把 Agent 接进现有系统且不失控”的工程能力。
- 接下来做项目时，优先找那种能体现 `状态机 + 工具调用 + 日志/评测` 的题，而不是只做一个会聊天的 demo。

## 今晚可验证动作（10-20分钟）

1. 读 ADK 2.0 文章，只回答一个问题：你手头最熟悉的一个业务流程，哪些步骤必须 deterministic，哪些步骤才该交给 LLM？
2. 再读 OpenAI Deployment Simulation，记一句自己的定义：它评估的不是“回答好不好”，而是“新模型在真实部署上下文里的行为分布会不会变差”。
3. 最后扫一眼 Cloudflare 的 AI traffic 方案，画一个 3 桶草图：`Search`、`Agent`、`Training`，想清楚未来你自己的 Agent 应该以哪一类身份访问外部站点。

## 原文链接

1. [OpenAI: Previewing GPT-5.6 Sol](https://openai.com/index/previewing-gpt-5-6-sol/)
2. [Anthropic: Introducing Claude Sonnet 5](https://www.anthropic.com/news/claude-sonnet-5)
3. [OpenAI: Predicting model behavior before release by simulating deployment](https://openai.com/index/deployment-simulation/)
4. [Google Developers Blog: Why we built ADK 2.0](https://developers.googleblog.com/why-we-built-adk-20/)
5. [Google Developers Blog: Driving the Agent Quality Flywheel from Your Coding Agent](https://developers.googleblog.com/driving-the-agent-quality-flywheel-from-your-coding-agent/)
6. [GitHub Changelog: Shape Copilot code review around your team](https://github.blog/changelog/2026-06-02-shape-copilot-code-review-around-your-team/)
7. [GitHub Changelog: Kimi K2.7 Code is generally available in GitHub Copilot](https://github.blog/changelog/2026-07-01-kimi-k2-7-is-now-available-in-github-copilot/)
8. [Vercel: AI SDK 7 is now available](https://vercel.com/blog/ai-sdk-7)
9. [Cloudflare: Your site, your rules: new AI traffic options for all customers](https://blog.cloudflare.com/content-independence-day-ai-options/)
10. [Cloudflare: Announcing the Monetization Gateway](https://blog.cloudflare.com/monetization-gateway/)
11. [Hugging Face: The Open Source Community is backing OpenEnv for Agentic RL](https://huggingface.co/blog/openenv-agentic-rl)

## 原文中文翻译链接（机器翻译）

1. [GPT-5.6 Sol 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/previewing-gpt-5-6-sol/)
2. [Claude Sonnet 5 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://www.anthropic.com/news/claude-sonnet-5)
3. [Deployment Simulation 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/deployment-simulation/)
4. [ADK 2.0 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://developers.googleblog.com/why-we-built-adk-20/)
5. [Agent Quality Flywheel 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://developers.googleblog.com/driving-the-agent-quality-flywheel-from-your-coding-agent/)
6. [Copilot code review + MCP 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.blog/changelog/2026-06-02-shape-copilot-code-review-around-your-team/)
7. [Kimi K2.7 Code 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.blog/changelog/2026-07-01-kimi-k2-7-is-now-available-in-github-copilot/)
8. [AI SDK 7 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://vercel.com/blog/ai-sdk-7)
9. [Cloudflare AI traffic 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://blog.cloudflare.com/content-independence-day-ai-options/)
10. [Monetization Gateway 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://blog.cloudflare.com/monetization-gateway/)
11. [OpenEnv 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://huggingface.co/blog/openenv-agentic-rl)
