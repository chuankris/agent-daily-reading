# 今日雷达主题

Agent 开发者工作面与执行后端正在合流

## 今日雷达总览（10条）

### 1. Google 把 Managed Agents 正式带进 Gemini API
摘要：5 月 19 日，Google 发布 Managed Agents，单次 API 调用即可拉起隔离 Linux 环境，让 Agent 直接推理、调工具、执行代码、浏览网页，并支持用 `AGENTS.md` / `SKILL.md` 定义可版本化能力。
为什么重要：这说明“自己搭 agent runtime”正在被平台层吃掉，差异点开始转向技能组织、状态恢复、权限边界和业务工具接入，而不是手写 loop。
原文链接：[Google 官方](https://blog.google/innovation-and-ai/technology/developers-tools/managed-agents-gemini-api/)
原文中文翻译链接（机器翻译）：[Google Translate](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://blog.google/innovation-and-ai/technology/developers-tools/managed-agents-gemini-api/)

### 2. Gemini 3.5 Flash 成为 Google Agent 栈的高速底座
摘要：Google 在 I/O 2026 开发者更新中称，Gemini 3.5 Flash 在多数基准上超过 Gemini 3.1 Pro，同时速度达到其他前沿模型的 4 倍，并作为 Antigravity / Managed Agents 的共优化底座。
为什么重要：对 Agent 工程来说，真正可用的不是“最强单轮模型”，而是“足够强且足够快”的执行底座；这会直接影响多步任务的总时延和成本。
原文链接：[Google 官方](https://blog.google/innovation-and-ai/technology/developers-tools/google-io-2026-developer-highlights/)
原文中文翻译链接（机器翻译）：[Google Translate](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://blog.google/innovation-and-ai/technology/developers-tools/google-io-2026-developer-highlights/)

### 3. Mistral Medium 3.5 + 云端远程编码 Agent 上线
摘要：5 月 22 日，Mistral 将 Mistral Medium 3.5 作为 Vibe 默认模型，并把编码 Agent 从本地终端推到云端异步运行，支持并行会话、跨 CLI/网页延续状态、完成后回推 PR。
为什么重要：这不是单纯换模型，而是“编码代理运行面”从本地 IDE 扩到远程可持续执行环境，和 OpenAI / Google 的方向高度一致。
原文链接：[Mistral 官方](https://mistral.ai/fr/news/vibe-remote-agents-mistral-medium-3-5/)
原文中文翻译链接（机器翻译）：[Google Translate](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://mistral.ai/fr/news/vibe-remote-agents-mistral-medium-3-5/)

### 4. Mistral Vibe 把工作 Agent 和编码 Agent 合成一个统一工作面
摘要：5 月 28 日，Mistral 把 Le Chat 演进为 Vibe，Work Mode 负责长任务，Code Mode 负责远程编码，并新增 VS Code 扩展，覆盖网页、IDE、终端三种入口。
为什么重要：开发者侧产品正在收敛成“一个对话面 + 多执行面”。这会改变你对 Agent 产品形态的理解：入口统一，执行后端分层。
原文链接：[Mistral 官方](https://mistral.ai/news/vibe-agent/)
原文中文翻译链接（机器翻译）：[Google Translate](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://mistral.ai/news/vibe-agent/)

### 5. Mistral 开源 Search Toolkit，把检索、摄取、评测并到一个框架
摘要：5 月 28 日，Mistral 发布 Search Toolkit 公测版，用统一接口覆盖 ingestion、retrieval、evaluation，支持 BM25、向量检索、混合检索，以及 recall / precision / MRR / NDCG 等指标。
为什么重要：这对 RAG 工程很关键。行业正在把“先搭检索、再补评测”的临时拼装，转成“把检索质量单独工程化管理”的标准流水线。
原文链接：[Mistral 官方](https://mistral.ai/news/search-toolkit/)
原文中文翻译链接（机器翻译）：[Google Translate](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://mistral.ai/news/search-toolkit/)

### 6. OpenAI Workspace Agents 加强企业接入和可运营性
摘要：OpenAI 5 月 28 日的 Enterprise / Edu 发布说明里，把 Workspace agents 扩展到 GPT-5.5、推理力度控制、Slack 线程回复、GitHub Enterprise/Snowflake/Databricks 模板，并明确支持 `managed MCP server URLs`。
为什么重要：企业 Agent 不再只是“会回答”，而是在往“可发布、可接入、可治理、可分析”的内部平台走。MCP 正在进入正式企业配置流。
原文链接：[OpenAI 官方](https://help.openai.com/en/articles/10128477-chatgpt-enterprise-edu-release-notes)
原文中文翻译链接（机器翻译）：[Google Translate](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://help.openai.com/en/articles/10128477-chatgpt-enterprise-edu-release-notes)

### 7. GitHub Copilot Business / Enterprise 默认基座切到 GPT-5.3-Codex，并给出 LTS
摘要：GitHub 5 月 17 日宣布，GPT-5.3-Codex 成为 Copilot Business / Enterprise 的默认模型，同时这是首个提供 12 个月可用期保证的 LTS 模型。
为什么重要：企业不会无限追新模型，真正影响生产采用的是“稳定窗口 + 审批窗口 + 生命周期承诺”。LTS 正在变成 Agent 时代的企业采购语言。
原文链接：[GitHub 官方](https://github.blog/changelog/2026-05-17-gpt-5-3-codex-is-now-the-base-model-for-copilot-business-and-enterprise/)
原文中文翻译链接（机器翻译）：[Google Translate](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.blog/changelog/2026-05-17-gpt-5-3-codex-is-now-the-base-model-for-copilot-business-and-enterprise/)

### 8. Anthropic 收购 Stainless，继续押注“Agent 连接层”
摘要：Anthropic 5 月 18 日宣布收购 Stainless，明确把它和 Claude Platform 的 developer experience、agent connectivity 绑定在一起。
为什么重要：这不是普通并购新闻，而是说明 API/SDK 生成、工具契约和连接体验正在成为平台护城河，和 MCP 一起构成下一阶段竞争点。
原文链接：[Anthropic 官方](https://www.anthropic.com/news/anthropic-acquires-stainless)
原文中文翻译链接（机器翻译）：[Google Translate](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://www.anthropic.com/news/anthropic-acquires-stainless)

### 9. MCP Java SDK 进入稳定线，同时 2.x 开始为规范演进做兼容
摘要：MCP Java SDK 在 5 月进入 1.x 稳定迭代，同时发布 2.0.0-M 系列，重点处理前后向兼容、JSON Schema 方言、默认工具输入校验、传输层细节等问题。
为什么重要：这对你的 Java 背景非常有用。JVM 不再只是“外围接入层”，已经可以作为正式 MCP client/server 落点，适合桥接遗留系统和 Agent 平台。
原文链接：[GitHub Releases](https://github.com/modelcontextprotocol/java-sdk/releases)
原文中文翻译链接（机器翻译）：[Google Translate](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/modelcontextprotocol/java-sdk/releases)

### 10. MCP 继续往 Stateless 方向收敛，`server/discover` 成为关键补丁
摘要：MCP 官方 SEP-2575 明确补上 discovery RPC，服务器必须实现 `server/discover`，客户端可在无会话假设下探测能力和版本，减轻实现分歧。
为什么重要：这场路线之争很实际。越偏 stateless，越容易做网关、代理、弹性扩容、跨语言桥接；越依赖 transport session，工程复杂度越高。
原文链接：[MCP 官方 SEP](https://modelcontextprotocol.io/seps/2575-stateless-mcp)
原文中文翻译链接（机器翻译）：[Google Translate](https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://modelcontextprotocol.io/seps/2575-stateless-mcp)

## 重点解读（1条）

### Mistral Search Toolkit：RAG 工程正在从“拼装检索链路”转向“可评测的检索平台”

这条我排今天的重点，不是因为它最热，而是因为它最适合你当前的转型路线。

Mistral 这次不是再发一个“更强 embedding”或“更强 reranker”，而是把检索系统最麻烦的三段一起收进统一框架：数据摄取、检索、评测。官方原文直接点出很多团队还在把 ingestion、retrieval、evaluation 分散到不同工具里，结果大部分工程时间都花在 plumbing，而不是质量提升。

这里最重要的不是“又一个搜索框架”，而是它把检索质量和生成质量拆开管理。官方明确给了 recall、precision、MRR、NDCG，并强调可以独立评测 retriever、横向比较配置、随语料演化持续追踪质量。对 Agent/RAG 落地来说，这非常关键，因为很多失败并不是模型不会答，而是：

1. 文档切分方式不对
2. 元数据抽取不稳定
3. 稀疏/稠密/混合检索没有针对语料调优
4. 每次改配置后没有一致的回归评测

这和你从 Java/IoT 集成转过来的经验高度契合。你原来的优势不是“写 prompt”，而是把多系统接起来、把链路稳定下来、把输入输出约束清楚。Search Toolkit 这类框架，正好把 Agent 工程里最像“集成工程”的部分放大了：解析器、索引流、评测集、版本回归、上线前验证。

更实用的一点是，它已经给了 starter app，启动路径很短：Docker + `uv`，先跑本地 Vespa，再 ingest 一份样例文本，再执行一次 search。这个门槛比从零搭一套可评测 RAG 基础设施低很多，适合你今晚就验证。

我的判断：未来一年，RAG 的门槛不会再是“能不能接向量库”，而是“你能不能把检索质量变成一条可回归、可发布、可解释的工程流水线”。Mistral 这次做的是这个方向。

## 对当前转型路线的影响

1. 先把“运行时”和“连接层”当成主战场，而不是只盯模型榜单。Managed Agents、Vibe、Workspace Agents 都在说明平台层已经开始标准化。
2. Java 背景不是包袱。MCP Java SDK 和 stateless MCP 的推进，反而让你更适合做企业系统桥接、协议适配、权限封装和工具治理。
3. RAG 要尽快从“demo 检索”升级到“可评测检索”。Search Toolkit 这类框架比继续堆 prompt 更值得投入时间。
4. 关注会话面和执行面的分离。统一入口、远程执行、可恢复状态、人工审批，会是之后多数 Agent 产品的默认结构。

## 今晚可验证动作（10-20分钟）

用 Mistral Search Toolkit 的 starter app 快速跑一遍最小链路：

1. 打开官方文章里的 starter app 仓库。
2. 在本机准备 Docker 和 `uv`。
3. 执行官方给出的 `uvx copier copy gh:mistralai/search-starter-app my-search-project`。
4. 进入项目后跑 `make setup-vespa`、`make ingest path=sample_data/hello.txt`、`make search query="hello world"`。
5. 重点不要纠结效果，先观察它把 ingest / search / eval 组织成什么目录和配置边界。

