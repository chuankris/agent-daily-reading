# 2026-04-30 Skill 专题：是什么、怎么写、怎么学（含开源仓库）

原文来源：

- 标题：Skills | openai/codex docs  
- 链接：https://github.com/openai/codex/blob/main/docs/skills.md  
- 来源类型：官方文档

- 标题：openai/skills（Skills Catalog for Codex）  
- 链接：https://github.com/openai/skills  
- 来源类型：官方仓库

- 标题：skill-creator (SKILL.md) | openai/skills  
- 链接：https://github.com/openai/skills/blob/main/skills/.system/skill-creator/SKILL.md  
- 来源类型：官方示例

为什么今天读这个：

你现在已经在用自动化学习和每日任务了，下一步很自然就是把“重复出现的工作方式”沉淀成 Skill。Skill 的价值不是多一个概念，而是把你自己的工程方法固定下来，让 Agent 每次都按你认可的流程做事。

原文翻译（先读这一段）：

官方文档把 Skill 定义为一组可复用的任务能力封装，通常以 `SKILL.md` 为核心，并可配套脚本、资源与参考材料。Skill 的目标是让 Agent 在特定场景下更稳定地执行流程，而不是每次都从零推理。官方仓库展示了多个可复用 Skill 模板，强调 Skill 应具备清晰触发条件、边界、步骤和输出预期，便于团队共享与迭代。

中文精读：

## 1) 什么是 Skill（工程视角）

你可以把 Skill 理解成：**给 Agent 的“可复用作业说明书”**。  
它跟普通 Prompt 的区别在于：

1. Prompt 常是一次性对话输入。  
2. Skill 是可反复调用的工作流约束。  
3. Skill 往往会绑定固定目录、脚本、数据源和输出规范。

换成你熟悉的 Java 工程类比：

1. Prompt 像一次临时 SQL。  
2. Skill 像沉淀后的 Service 模板 + SOP + 运行脚本。

## 2) Skill 应该怎么定义（最小可用结构）

初学者最稳的写法是把 `SKILL.md` 固定成 5 块：

1. `description`  
- 这个 Skill 用来解决什么问题。

2. `use when`  
- 什么场景应该调用它。  
- 什么场景不该调用它。

3. `inputs`  
- 需要哪些输入。  
- 输入缺失时怎么处理。

4. `workflow`  
- 固定 4 到 8 步，保持短闭环。  
- 每步写动作和产物。

5. `output contract`  
- 输出结构和验收标准。  
- 失败时的回退动作。

你后面做 ToB Agent 特别建议加两个块：

1. `risk control`（权限、写操作确认、审计）  
2. `observability`（日志字段、trace 关键点）

## 3) 初学者怎么上手编写 Skill（建议路径）

建议你按这个顺序来，成功率最高：

1. 先挑一个你每周重复做 3 次以上的任务。  
- 例如：代码变更检查、接口联调 checklist、日报生成。

2. 先写“人工版流程”，再转 `SKILL.md`。  
- 不要一上来追求复杂自动化。

3. 第一个 Skill 只管 1 件事。  
- 不要把“需求分析+编码+测试+部署”全塞进去。

4. 用 5 到 10 个真实样例跑回归。  
- 看是否稳定触发。  
- 看输出是否可直接消费。

5. 每周只改一个维度。  
- 这一周改触发条件。  
- 下一周再改输出结构。  
- 避免一口气全改，难以定位效果。

## 4) Skill 开发里的常见坑和难点

1. 坑：描述太泛  
- 表现：Agent 经常误触发或不触发。  
- 解法：`use when` 写成可判定条件。

2. 坑：步骤太长  
- 表现：中途漂移、重复、漏步骤。  
- 解法：单个 Skill 控制在短流程，复杂任务拆多个 Skill。

3. 坑：没有输出契约  
- 表现：下游系统解析困难。  
- 解法：输出字段固定化，必要时强制结构化格式。

4. 坑：忽略失败路径  
- 表现：工具超时或鉴权失败后流程断掉。  
- 解法：在 `workflow` 里显式写重试、降级、人工接管。

5. 坑：没有版本意识  
- 表现：团队共用后行为不一致。  
- 解法：Skill 文件版本化，变更要有回归样本和说明。

## 5) 你可以直接学习的开源仓库

优先学习这几个（先官方、再生态）：

1. `openai/codex`  
- https://github.com/openai/codex  
- 看点：官方 `docs/skills.md`，理解 Skill 在 Codex 中的推荐使用方式。

2. `openai/skills`  
- https://github.com/openai/skills  
- 看点：大量可安装的 Skill 目录结构、系统 Skill、实验 Skill。

3. `openai/skills` 的 `skill-creator`  
- https://github.com/openai/skills/blob/main/skills/.system/skill-creator/SKILL.md  
- 看点：如何把“创建 Skill”本身写成一个 Skill，适合你照着仿写第一版模板。

扩展阅读（社区索引类）：

1. `skillmatic-ai/awesome-agent-skills`  
- https://github.com/skillmatic-ai/awesome-agent-skills  
- 价值：收集了多平台 Skill 生态，但质量参差，建议用于“找方向”，不要盲信全部实现细节。

你只需要记住的 3 个点：

1. Skill 是“流程能力封装”，不是“更长的 Prompt”。  
2. 初学者先做短流程单任务 Skill，先求稳触发和稳输出。  
3. 判断 Skill 好坏的核心是：能否重复、可否观测、是否可回归。

和 Agent 工程的关系：

你后面要做的不只是“会调模型”，而是“把团队经验做成可复用能力”。Skill 正是这一步的桥：把你十年工程经验固化为 Agent 可执行的标准流程。

今晚可动手：

写一个你自己的第一版 Skill 草稿（只做 1 件事），按下面问题检查：

1. 这个 Skill 的触发条件是不是可判定的？  
2. 这个 Skill 的输出能不能被下游系统直接消费？

原文关键段落翻译（人工翻译，放在文末）：

1. Skill 的目标是把高频任务流程标准化，让 Agent 在相似任务上表现更稳定，而不是每次临时生成做法。  
2. 一个可维护的 Skill 应该明确“何时使用、如何执行、输出什么”，并尽量减少歧义。  
3. Skill 可以和脚本、资源文件协同工作，用于把复杂工作流拆成可复用能力单元。  
4. 团队共享 Skill 时，结构清晰与边界定义通常比“写得多”更重要。

原文中文翻译链接（机器翻译）：

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/openai/codex/blob/main/docs/skills.md
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/openai/skills
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://github.com/openai/skills/blob/main/skills/.system/skill-creator/SKILL.md

