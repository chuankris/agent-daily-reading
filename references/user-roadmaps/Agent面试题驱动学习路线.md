# Agent面试题驱动学习路线

更新时间：2026-04-26
目的：不是继续堆更多题，而是把 `agent面试题目.md` 反过来变成学习路线图。核心思路是：先看高频题，再决定短期、中期、长期应该学什么、读什么、做到什么程度。

## 1. 先说结论

你现在最需要的，不是先把题库扩到 100 题，而是建立这条链路：

面试题 -> 核心概念 -> 官方文章/论文 -> 最小项目 -> 口头表达

因为真正的面试通过，不取决于你“背了多少题”，而取决于：
- 你能不能把题目讲清楚
- 你能不能用项目证明你真的做过
- 你能不能解释设计取舍与问题排查

所以后续学习建议分成 3 段：
1. 短期：先把高频基础题答顺，建立最小可运行 Agent 项目
2. 中期：补齐系统化能力，形成评测、可视化、工程化认知
3. 长期：冲平台深度与差异化能力，靠作品拉开差距

## 2. 短期目标（0-6周）

### 2.1 目标定位

短期不是追求“很厉害”，而是做到两件事：
- 面试遇到 Agent 基础题，不会卡住
- 手里有一个最小项目，不至于只停留在概念层

### 2.2 这一阶段要重点覆盖的面试题

优先吃透这些题：
- 1-5：Agent 是什么、单多 Agent、组件、ReAct、死循环
- 8-10：上下文工程、压缩时机、信息选择
- 16-20：Function Calling、MCP、Tool Router、失败处理、Skill vs API
- 29-31：Claude Code / Cursor / Copilot 的正确使用方式

### 2.3 这个阶段必须理解的概念

1. Workflow 和 Agent 的区别
2. Tool Calling / Function Calling 基本流程
3. JSON Schema 和结构化输出
4. Prompt 约束与上下文最小化
5. Agent 的基本执行环：输入 -> 思考 -> 调工具 -> 反馈 -> 结束
6. 为什么 Agent 会死循环、失控、漂移
7. AI 编码工具不是代写，而是结对工程师

### 2.4 必读文章 / 文档

1. Anthropic: Building Effective Agents
- 作用：建立 Agent 与 Workflow 的正确认知，理解什么时候该用 Agent
- 链接：[Building Effective Agents](https://www.anthropic.com/research/building-effective-agents/)

2. OpenAI Function Calling Guide
- 作用：理解 Tool Calling 的标准流程、schema、tool call output
- 链接：[Function calling](https://platform.openai.com/docs/guides/function-calling/introduction)

3. OpenAI Help: Function Calling in the API
- 作用：快速把 Structured Outputs、Tools、Agents 平台串起来
- 链接：[Function Calling in the OpenAI API](https://help.openai.com/en/articles/8555517-function-calling-in-the-openai-api)

4. OpenAI Agents SDK Overview
- 作用：理解 Agent、tool、handoff、trace 这些工程对象
- 链接：[Agents SDK](https://platform.openai.com/docs/guides/agents-sdk/)

5. MCP Official Specification Overview
- 作用：建立 MCP 的正式定义，别只停留在二手解释
- 链接：[Model Context Protocol Specification](https://modelcontextprotocol.io/specification/draft)

### 2.5 短期产出要求

你至少要做出一个最小项目：
- Python + FastAPI 的 Agent 服务
- 至少 1 个 tool
- 至少 1 个可回看的 trace/log
- 至少 1 组测试

### 2.6 短期达标标准

达到下面状态，就说明短期完成得不错：
- 你能口头讲清 Agent 和 Workflow 的区别
- 你能解释 Function Calling 和 MCP 的区别
- 你能自己写一个 tool schema，并处理异常
- 你有一个可运行的小项目，不是只有笔记

## 3. 中期目标（2-4个月）

### 3.1 目标定位

中期的核心是从“会做一个 Agent Demo”，进化到“理解一个 Agent 系统为什么能跑起来、为什么会出问题、怎么评估好坏”。

### 3.2 这一阶段重点覆盖的面试题

优先吃透这些题：
- 6-7：短期记忆、长期记忆、检索维度
- 11-15：RAG、混合检索、chunk、引用、评估指标
- 21-25：评测平台、轨迹回放、稳定性、可观测性、CI/CD
- 32-35：Redis、缓存、慢查询、并发与幂等
- 36-38：系统设计开放题

### 3.3 这个阶段必须理解的概念

1. Short-term memory / long-term memory 的职责边界
2. RAG 链路：query rewrite -> retrieve -> rerank -> construct -> answer
3. chunk、metadata、citation、hallucination control
4. 评测是什么，不只是“跑一下感觉不错”
5. step 级日志、trace、回放为什么重要
6. 稳定性治理：超时、重试、限流、熔断、降级、幂等
7. 前端为什么要做轨迹可视化，而不是只有后端接口

### 3.4 必读文章 / 文档

1. ReAct 论文
- 作用：理解边思考边行动的经典范式
- 链接：[ReAct paper](https://arxiv.org/abs/2210.03629)

2. 原始 RAG 论文
- 作用：建立 RAG 的第一性原理，不被各种包装概念带偏
- 链接：[Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)

3. CRAG 论文
- 作用：理解检索质量差时的纠错思路
- 链接：[Corrective Retrieval-Augmented Generation](https://arxiv.org/abs/2401.15884)

4. LangGraph Overview
- 作用：建立有状态、多步、可控 Agent 编排的工程视角
- 链接：[LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview)

5. FastAPI 官方 Tutorial
- 作用：把你的 Agent 服务写得更像工程项目而不是脚本
- 链接：[FastAPI Tutorial](https://fastapi.tiangolo.com/tutorial/)

6. Docker 官方 Get Started
- 作用：完成服务容器化的基本能力
- 链接：[Docker Get Started](https://docs.docker.com/get-started/)

### 3.5 中期产出要求

你应该做出一个“像样”的项目 v1：
- 后端：Agent 服务 + tool calling + memory/rag 中的一项
- 前端：轨迹列表/详情页面
- 平台：基础评测脚本，能批量执行任务
- 工程化：Docker 化 + README + 测试

### 3.6 中期达标标准

- 你能讲清一次完整 Agent 请求链路
- 你能解释为什么需要评测、看哪些指标
- 你能说出一个真实失败案例，以及怎么排查
- 你能演示一个轨迹回放页面或日志分析过程

## 4. 长期目标（4-12个月）

### 4.1 目标定位

长期目标不是“再多学几个框架”，而是补齐大厂最看重的平台深度与差异化能力，让你从“能做应用”升级到“能做 Agent 基础设施”。

### 4.2 这一阶段重点覆盖的面试题

优先深挖这些题：
- 21-28：评测、回放、可观测、CI/CD、Docker/K8s、成本控制
- 36-40：系统设计、稳定性、线上事故排查、项目深挖
- 再额外补：MCP 实现、协议适配、Rust、沙箱与权限隔离

### 4.3 这个阶段必须理解的概念

1. Agent 平台化：统一接入、统一 trace、统一评测、统一发布
2. 容器化运行时与环境隔离
3. Kubernetes 的部署、扩缩容、健康检查、配置管理
4. 成本、延迟、成功率三者之间的权衡
5. 多 Agent / 多工具并发带来的状态一致性问题
6. Rust 在工具运行时、高性能模块、隔离场景中的价值
7. 安全：权限控制、沙箱、工具白名单、审计

### 4.4 必读文章 / 文档

1. OpenAI Agents SDK 新能力介绍（2026-04-15）
- 作用：理解长任务、沙箱执行、文件/命令级 agent 基础设施方向
- 链接：[The next evolution of the Agents SDK](https://openai.com/index/the-next-evolution-of-the-agents-sdk)

2. MCP Overview / Base Protocol
- 作用：深入理解生命周期、能力协商、client/server feature
- 链接：[MCP Overview](https://modelcontextprotocol.io/specification/2025-06-18/basic/index)

3. Kubernetes Basics
- 作用：完成容器平台认知闭环
- 链接：[Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/)

4. OpenTelemetry Documentation
- 作用：补可观测性基础，帮助你理解 trace/log/metrics
- 链接：[OpenTelemetry Docs](https://opentelemetry.io/docs/)

5. Rust Book
- 作用：面向工程实战地补 Rust，而不是只背语法
- 链接：[The Rust Programming Language](https://doc.rust-lang.org/book/)

### 4.5 长期产出要求

你最好形成 3 个可展示成果：
- 主项目：Agent 评测与轨迹平台
- 工程项目：Docker/K8s/CI/CD/可观测性完整闭环
- 差异化项目：MCP 适配层、Rust 工具运行时或沙箱模块

### 4.6 长期达标标准

- 你能用项目证明自己不只是会调模型 API
- 你能回答“为什么这样设计、出了问题怎么查、指标怎么衡量”
- 你有至少一个能体现平台思维的项目，而不是只有业务页面

## 5. 文章怎么读，才不会白读

建议每篇文章都按这个方式处理：

1. 第一次阅读
- 只回答：这篇文章解决什么问题？核心结论是什么？

2. 第二次阅读
- 画出流程图：输入、决策、执行、反馈、失败点分别在哪里

3. 第三次阅读
- 映射到面试题：这篇文章能回答 `agent面试题目.md` 里的哪些问题？

4. 最后落地
- 至少把其中一个概念变成代码、配置或实验结果

## 6. 你现在最应该补的顺序

如果按当前优先级排序，我建议你这样推进：

1. 先补短期
- Agent 基础、Function Calling、MCP、上下文工程、AI coding 工具工作流

2. 再补中期
- Memory、RAG、评测、轨迹、日志、Docker、前端可视化

3. 最后补长期
- K8s、Rust、MCP 深水区、可观测性、平台化和安全隔离

## 7. 最重要的一句话

你后面所有学习，不要再按“语言目录”来学，而要按“面试问题簇”来学。

比如：
- 今天不是学 Python，而是在解决“Tool Calling 怎么设计更稳”
- 这周不是学 Docker，而是在解决“Agent 服务怎么交付成可运行系统”
- 这个月不是学 RAG，而是在解决“检索增强系统怎么评估和解释”

这样学，最后才会真的和岗位要求对齐。

---

补充说明：本文档中的文章优先选择官方文档、官方规范和经典论文，目的是让你建立一手理解，减少被二手教程带偏的概率。
