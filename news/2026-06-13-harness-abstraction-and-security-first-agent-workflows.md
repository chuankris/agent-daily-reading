# 2026-06-13 Agent 工程雷达：harness 抽象开始成形，安全先行工作流加速落地

## 今日雷达总览（10条）

1. **Anthropic 在 2026-06-09 发布 Claude Fable 5 与 Claude Mythos 5**
摘要：Fable 5 被定位为可普遍使用的 Mythos 级模型，强调更强的长程自主执行、代码、视觉、科研与记忆能力；Mythos 5 则先通过 Project Glasswing 面向受信任的防守方开放。
为什么重要：这不是单纯的“模型更强”新闻，而是明确把“公开可用能力”和“高风险能力受限开放”分层了。以后做 Agent 选型时，模型能力、可用边界、以及安全降级路径要一起看。

2. **Vercel 在 2026-06-12 为 AI SDK 7 引入 HarnessAgent**
摘要：Vercel 把 Claude Code、Codex、Pi 这类现成 agent harness 抽成统一 API，主张“先写 agent 逻辑，再按需切换 harness”，而不是把应用逻辑绑死在某个工作台或某个 CLI 上。
为什么重要：这条最像 Agent 工程里的“接口层标准化”信号。你后面无论做内部代码助手、故障排查 agent，还是半自动开发流，真正稳定的资产应该是任务编排、权限边界、上下文装配，而不是某一家交互壳子。

3. **GitHub 在 2026-06-11 将 Agentic Workflows 推到 public preview**
摘要：GitHub 允许你用自然语言 Markdown 定义 issue triage、CI 失败分析、文档更新等推理型任务，再编译成标准 GitHub Actions YAML，在现有 runner 和策略体系里运行。
为什么重要：这说明 agent workflow 正在从“聊天入口”迁到“代码仓库原生自动化入口”。对工程人来说，能进 Actions、吃到现有审计和权限体系，比另起一个智能平台更实际。

4. **GitHub 在 2026-06-12 给 Copilot code review 增加组织级控制与内容排除**
摘要：Copilot code review 现在支持组织级 runner 默认值与锁定、内容排除规则继承，以及移除仓库自定义指令字符限制。
为什么重要：Agent 进代码审查链路后，真正卡住落地的往往不是能力，而是“哪些文件能看、在哪个 runner 跑、组织策略怎么统一”。这次更新是典型的企业级落地补丁。

5. **GitHub 在 2026-06-11 为 Copilot CLI 上线统一 `/settings`**
摘要：Copilot CLI 把原本分散在 `/theme`、`/experimental` 等命令和手工配置文件里的选项统一到 schema 驱动的 `/settings` 入口，支持全屏 UI、单行设置和重置。
为什么重要：CLI agent 正在从“黑箱工具”变成“可配置工作台”。这会直接影响团队里如何沉淀统一配置、实验开关和默认行为，尤其适合你后面做可复用 agent profile。

6. **GitHub 在 2026-06-10 为 Copilot CLI 增加 `/security-review`**
摘要：新命令会直接审查本地改动，返回高置信度安全问题、严重度和可操作建议，覆盖注入、XSS、路径遍历、弱加密等常见高影响漏洞类型。
为什么重要：这让“先本地过一遍 AI 安全审查，再提交代码”开始像一个真实工作流，而不只是营销口号。对转型 Agent 工程来说，安全 review 也在变成默认链路的一部分。

7. **OpenAI 在 2026-06-10 宣布可通过 Oracle 云承诺采购 OpenAI 模型与 Codex**
摘要：OCI 客户未来几周可以把合格的 Oracle Universal Credits 用到 OpenAI 模型和 Codex 上，不必再走新的采购通道。
为什么重要：很多 Agent 项目不是死在技术，而是死在采购、合规和预算归属。OpenAI 这步是在补“企业进生产”的最后几公里，意味着 Codex 类能力会更容易进入传统大组织。

8. **Anthropic 在 2026-06-02 扩展 Project Glasswing 到约 150 家新组织**
摘要：Anthropic 表示首批合作方已用 Claude Mythos Preview 找到超过 1 万个高危或严重漏洞，现在将项目扩展到 15 个以上国家的新组织，并覆盖电力、供水、医疗、通信、硬件等行业。
为什么重要：这条说明安全 Agent 已经不只是软件公司内部实验，而是在走关键基础设施路线。对你来说，这也是一个很现实的切入方向：Agent 工程和可观测、安全、审计天然相连。

9. **Ollama 在 2026-06-11 更新 MLX 引擎，强化 Apple Silicon 本地运行**
摘要：新版本更重度利用 Apple 统一内存和 Metal/MLX，提升响应速度、首 token 时间和内存效率，还支持 NVFP4 这类更高质量的量化格式。
为什么重要：这继续降低“本地可跑 agent”的门槛。你如果后面想做内网、离线、私有代码仓助手，硬件现实性比榜单分数更关键，这类 runtime 改进值得持续跟。

10. **Hugging Face 在 2026-06-12 开源 `serge`，做 GitHub 原生 AI Code Review**
摘要：`serge` 是一个开源 PR reviewer，可用任意 OpenAI-compatible 模型，通过 GitHub Action、GitHub App 或 Web 应用运行，并把 review policy 放在仓库里。
为什么重要：这类项目最适合转型期拿来拆。它把“仓库规则、评论发布、人工复核、安全边界”这些真实工程问题都摊开了，比看纯 demo agent 更能学到可落地结构。

## 重点解读（1条）

### 1. Vercel 的 HarnessAgent，为什么值得你重点跟

这条最值得今天重点看，因为它击中了一个越来越清晰的路线判断：**Agent 工程的核心抽象，正在从“模型调用”上移到“harness 调用”。**

过去很多人做 agent，默认思路是：
- 选一个模型
- 套一个框架
- 加若干工具
- 再在某个产品壳子里运行

但 Vercel 这次的说法更进一步。它不是只统一模型接口，而是统一更上层的那一层：`skills`、`permissions`、`sandbox`、`sessions`、`sub-agents`、`runtime configuration`。也就是说，真正决定 agent 行为质量的，不只是底座模型，而是模型外面那整套执行壳。

这对你特别重要，因为你未来大概率不会停留在“写一个会调 API 的 demo agent”。你更可能要面对这些问题：
- 同一个任务要不要换不同执行壳，比如 Codex、Claude Code、内部 agent？
- 哪些工具能调用，哪些文件能写，哪些命令需要审批？
- 会话状态怎么保存，失败后怎么恢复，日志怎么回放？
- 子 agent 怎么拆，验证步骤怎么固定，输出格式怎么约束？

这些问题本质都不在 prompt 层，而在 harness 层。

如果这个方向继续成立，后面会有两个直接后果：

第一，**工程资产会从“prompt 模板”转向“可迁移的 harness 规范”。**  
真正值钱的，不再是某一句神奇提示词，而是你怎样定义任务边界、权限模型、上下文注入、验证顺序和失败恢复。

第二，**不同厂商 Agent 之间的切换成本会下降，但前提是你先把自己的任务协议写清楚。**  
这和你从 Java/IoT 集成转过来的经验很像。稳定系统靠的不是某一个 SDK，而是清晰接口、状态约束、异常路径和审计链路。Agent 工程也正在变成同一类问题。

所以这条新闻最值得内化的不是 “Vercel 又发了一个 SDK 功能”，而是这个原则：

**以后做 Agent，先设计 harness，再谈模型切换；先定义执行边界，再谈智能程度。**

## 对当前转型路线的影响

- 近期学习重心可以继续压在 `MCP/工具接入 + sandbox/权限边界 + eval/验证链路 + 可观测执行日志`，这四块的耦合正在明显增强。
- 如果你要做作品集，优先做“可切换执行壳”的 agent，而不是只绑定某个单一模型或单一 CLI 的 agent。
- 你过去做系统集成时积累的接口、协议、状态机、异常处理经验，会越来越像 Agent 工程里的核心能力，而不是包袱。
- 未来 2 到 3 周可以考虑做一个小型 demo：同一任务同时适配两种 harness，比较上下文装配、权限控制、验证输出和失败恢复差异。

## 今晚可验证动作（10-20分钟）

1. 选一个你熟悉的小仓库，写下一个固定任务，例如“读取报错、修改代码、运行测试、输出变更说明”。
2. 把这个任务拆成 5 个 harness 维度：`上下文`、`工具`、`权限`、`验证`、`日志`。
3. 每个维度只写 1 到 2 条明确规则，整理成一个最小 agent contract。
4. 最后补一句判断：如果明天把底层模型或 CLI 换掉，哪几条规则还能不变；那部分就是你真正该沉淀的工程资产。

## 原文链接

- 1. https://www.anthropic.com/news/claude-fable-5-mythos-5
- 2. https://vercel.com/changelog/program-agent-harnesses-with-ai-sdk
- 3. https://github.blog/changelog/2026-06-11-github-agentic-workflows-is-now-in-public-preview/
- 4. https://github.blog/changelog/2026-06-12-copilot-code-review-new-configurations-and-controls/
- 5. https://github.blog/changelog/2026-06-11-copilot-cli-configure-everything-from-one-place-with-settings/
- 6. https://github.blog/changelog/2026-06-10-dedicated-security-review-command-now-available-in-copilot-cli/
- 7. https://openai.com/index/openai-on-oracle-cloud/
- 8. https://www.anthropic.com/news/expanding-project-glasswing
- 9. https://ollama.com/blog/mlx-performance
- 10. https://github.com/huggingface/serge

## 原文中文翻译链接（机器翻译）

- 1. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://www.anthropic.com/news/claude-fable-5-mythos-5
- 2. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://vercel.com/changelog/program-agent-harnesses-with-ai-sdk
- 3. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.blog/changelog/2026-06-11-github-agentic-workflows-is-now-in-public-preview/
- 4. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.blog/changelog/2026-06-12-copilot-code-review-new-configurations-and-controls/
- 5. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.blog/changelog/2026-06-11-copilot-cli-configure-everything-from-one-place-with-settings/
- 6. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.blog/changelog/2026-06-10-dedicated-security-review-command-now-available-in-copilot-cli/
- 7. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://openai.com/index/openai-on-oracle-cloud/
- 8. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://www.anthropic.com/news/expanding-project-glasswing
- 9. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://ollama.com/blog/mlx-performance
- 10. https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/huggingface/serge
