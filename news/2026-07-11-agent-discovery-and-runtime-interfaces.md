# 今日雷达总览（10条）

### 1. OpenAI 于 2026-07-10 发布 GPT-5.6，并把多档位模型与并发子代理能力直接产品化
- 简述：`GPT-5.6 Sol / Terra / Luna` 同时上线；Responses API 还加入了 `Programmatic Tool Calling` 和 beta 阶段的 `Multi-agent`。
- 为什么重要：这不只是模型变强，而是把“模型分层 + 工具编排 + 子任务并发”一起下沉到平台层。以后做 Agent，不该默认所有任务都打到同一个旗舰模型。

### 2. Anthropic 于 2026-06-30 发布 Claude Sonnet 5，继续巩固“默认执行模型”档位
- 简述：Claude Sonnet 5 覆盖 Claude、Claude Code 和 Claude Platform，强调更强的多步执行、调试和工具使用能力，并给出比 Opus 更低的入口价格。
- 为什么重要：生产里的主力模型往往不是“最强”，而是“够强、够稳、够便宜”。Sonnet 5 是你理解模型选型和成本控制的典型样本。

### 3. Google 在 I/O 2026 开发者主题演讲中把 Gemini 3.5 与 Antigravity 放到同一条 Agent 主线
- 简述：Google 明确把 Gemini 3.5 系列模型和 Antigravity 这个 agent-first 开发平台一起推进，强调从“辅助”转向“可独立跨工作流行动的代理”。
- 为什么重要：模型能力、开发入口和运行时平台正在一体化。以后学习重点不只是模型 API，而是模型如何嵌进完整开发闭环。

### 4. Google 于 2026-06-20 公布 Agentic Resource Discovery（ARD）规范，开始补“能力发现层”
- 简述：ARD 试图标准化工具、技能、Agent 的发布、发现和验证，让 Agent 能跨组织找到并安全连接能力。
- 为什么重要：MCP、A2A 解决的是“怎么调”，ARD 解决的是“先怎么找到、怎么验证”。这会越来越像企业集成里的服务注册与能力目录。

### 5. Google 于 2026-06-27 演示 ADK + A2A 的跨语言多 Agent 协作，Java 路线价值被进一步放大
- 简述：Google 展示了如何用 A2A 让不同语言实现的 Agent 协作，并通过 `RemoteA2aAgent` 把远端 A2A 服务封装成本地子代理。
- 为什么重要：这直接利好有 Java/集成背景的人。你不必把一切重写成 Python，完全可以把跨语言 Agent 网络当成下一代集成总线来理解。

### 6. Vercel 于 2026-06-25 发布 AI SDK 7，Agent 常见 plumbing 继续标准化
- 简述：AI SDK 7 新增 reasoning control、tool/runtime context、provider files、skills support、MCP Apps 和 terminal UI 等能力。
- 为什么重要：这类 SDK 正在把 Agent 工程里最重复的胶水层抽象出来。你越早理解这些抽象，越容易搭出能迁移、能替换模型供应商的工程骨架。

### 7. Cloudflare 于 2026-07-01 把 Agents SDK 底层能力开放给 Flue 等框架，明确 `framework -> harness -> runtime` 三层栈
- 简述：Cloudflare 公开把 durable execution、durable filesystem、dynamic workflows 等运行时原语下沉到 Agents SDK，Flue 1.0 Beta 成为首批直接构建其上的开源框架。
- 为什么重要：生产级 Agent 已经不再是“提示词 + 几个工具”。真正难的是状态、执行恢复、并发和长期任务治理，这正是运行时层的价值。

### 8. GitHub 于 2026-07-02 上线 Copilot agent session streaming 公测，把 Agent 运行轨迹正式纳入可观测面
- 简述：企业云用户现在可以通过 streaming endpoint 和 REST API 访问 Copilot agent session 数据，覆盖 cloud agents、CLI、VS Code、Visual Studio 等入口。
- 为什么重要：这意味着 Agent 不再是黑盒。日志流、会话追踪和平台治理会越来越接近传统分布式系统的观测方式。

### 9. GitHub 于 2026-07-01 在 Copilot 中上线 Kimi K2.7 Code，首次把 open-weight 编码模型放进主流开发工作流
- 简述：Kimi K2.7 Code 作为首个 Copilot model picker 中可选的 open-weight 模型，先向 Pro、Pro+、Max 计划逐步开放。
- 为什么重要：这释放了一个清晰信号：主流编码 Agent 入口开始认真接纳开放权重模型。后续的模型路由、BYOM/BYOK 和成本分层会更快普及。

### 10. LangChain 于 2026-07-10 发布 OpenWiki Brains，把“主动记忆”做成可落地的 Agent 开源部件
- 简述：OpenWiki Brains 让 Agent 能从 Gmail、Notion、Git 仓库、Hacker News、Web search 等来源主动构建并持续更新本地 wiki 记忆。
- 为什么重要：很多 Agent 失败不是因为推理不够，而是上下文长期失忆。这个项目把“记忆工程”从 prompt 技巧推进到可维护的系统部件。

## 重点解读（1条）

### ARD：为什么“能力发现层”可能是 2026 下半年最容易被低估的 Agent 基础设施

过去几个月，大家更多在谈 MCP、A2A、工具调用、子代理并发，但这些都默认了一件事：**能力已经被你知道、并且已经接好了**。ARD 要补的是更靠前的一层，也就是 Agent 如何像调用服务注册中心一样，先发现可用能力、理解它的元数据，再决定是否连接。

这件事对工程落地的意义很大：

- 第一，它把 Agent 生态从“手工配置工具列表”推向“可搜索、可验证、可治理的能力目录”。
- 第二，它会把权限、发布者身份、版本声明、兼容协议这些传统平台治理问题重新带回 Agent 工程。
- 第三，它和你熟悉的 Java/IoT 集成思维非常接近。你过去做的是设备、服务、协议与权限的连接；现在只是把“设备目录”换成了“Agent/skill/tool 目录”。

如果 ARD 真正铺开，后面企业内部很可能会出现一类新基础设施：统一维护 `ai-catalog`，向内部 Agent 暴露 MCP servers、A2A endpoints、skills、审批能力和审计元数据。那时竞争力就不在于“会不会接一个模型”，而在于你能不能把这些能力组织成可靠的发现与接入平面。

对你来说，这条线值得优先跟，因为它天然连接你的旧优势：注册中心、协议适配、权限边界、服务治理、审计追踪。这些在 Agent 工程里不会消失，只会换个名字回来。

## 对当前转型路线的影响

- 学习主线可以进一步收敛到 5 块：`模型路由`、`能力发现`、`跨语言协作`、`运行时治理`、`观测与评测`。
- 你的 Java/IoT 集成经验不是包袱，反而是优势，因为 Agent 平台正在重新强调目录、编排、状态、权限和审计。
- 后续练手项目优先做那种同时包含 `MCP/A2A 接入 + 状态持久化 + 日志追踪 + 小型能力目录` 的系统，不要只停留在单轮聊天 demo。

## 今晚可验证动作（10-20分钟）

1. 先读一遍 ARD 公告，只回答一个问题：如果你给未来自己的 Agent 平台做一个最小能力目录，最少要登记哪些字段？
2. 用熟悉的 Java 或 Python 手写一个 `ai-catalog.json` 草稿，至少包含：`name`、`type`、`protocol`、`auth`、`version`、`owner`、`risk_level`、`endpoint`。
3. 再对照 ADK + A2A 的跨语言例子，补一条规则：哪些能力允许直接自动发现，哪些必须经过人工审批后才能接入。

## 原文链接

1. [OpenAI: GPT-5.6](https://openai.com/index/gpt-5-6/)
2. [Anthropic: Introducing Claude Sonnet 5](https://www.anthropic.com/news/claude-sonnet-5)
3. [Google Developers Blog: All the news from the Google I/O 2026 Developer keynote](https://developers.googleblog.com/all-the-news-from-the-google-io-2026-developer-keynote/)
4. [Google Developers Blog: Announcing the Agentic Resource Discovery specification](https://developers.googleblog.com/announcing-the-agentic-resource-discovery-specification/)
5. [Google Developers Blog: Build Cross-Language Multi-Agent Team with Google's Agent Development Kit and A2A](https://developers.googleblog.com/build-cross-language-multi-agent-team-with-google-agent-development-kit-and-a2a/)
6. [Vercel: AI SDK 7 is now available](https://vercel.com/blog/ai-sdk-7)
7. [Cloudflare Blog: Bringing more agent harnesses and frameworks to Cloudflare, starting with Flue](https://blog.cloudflare.com/agents-platform-flue-sdk/)
8. [GitHub Changelog: Copilot agent session streaming is now in public preview](https://github.blog/changelog/2026-07-02-copilot-agent-session-streaming-is-now-in-public-preview/)
9. [GitHub Changelog: Kimi K2.7 Code is generally available in GitHub Copilot](https://github.blog/changelog/2026-07-01-kimi-k2-7-is-now-available-in-github-copilot/)
10. [LangChain: Introducing OpenWiki Brains, general-purpose wiki memory for agents](https://www.langchain.com/blog/introducing-openwiki-brains-general-purpose-wiki-memory-for-agents)

## 原文中文翻译链接（机器翻译）

1. [GPT-5.6 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/gpt-5-6/)
2. [Claude Sonnet 5 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://www.anthropic.com/news/claude-sonnet-5)
3. [Google I/O 2026 开发者主题演讲中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://developers.googleblog.com/all-the-news-from-the-google-io-2026-developer-keynote/)
4. [ARD 规范中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://developers.googleblog.com/announcing-the-agentic-resource-discovery-specification/)
5. [ADK + A2A 跨语言多 Agent 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://developers.googleblog.com/build-cross-language-multi-agent-team-with-google-agent-development-kit-and-a2a/)
6. [AI SDK 7 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://vercel.com/blog/ai-sdk-7)
7. [Cloudflare Flue / Agents SDK 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://blog.cloudflare.com/agents-platform-flue-sdk/)
8. [Copilot agent session streaming 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.blog/changelog/2026-07-02-copilot-agent-session-streaming-is-now-in-public-preview/)
9. [Kimi K2.7 Code in Copilot 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.blog/changelog/2026-07-01-kimi-k2-7-is-now-available-in-github-copilot/)
10. [OpenWiki Brains 中文翻译](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://www.langchain.com/blog/introducing-openwiki-brains-general-purpose-wiki-memory-for-agents)
