# title

2026-04-30 为什么 Agent 工程先把 `pyproject.toml` 和 `pytest` 测试布局搭对

## original source

- 标题：Writing your `pyproject.toml` | Python Packaging User Guide
- 链接：https://packaging.python.org/en/latest/guides/writing-pyproject-toml/
- 来源类型：官方文档
- 发布时间：页面显示最近一个月内更新；2026-04-30 访问

- 标题：Good Integration Practices | pytest documentation
- 链接：https://docs.pytest.org/en/latest/explanation/goodpractices.html
- 来源类型：官方文档
- 发布时间：页面未标注首发时间；2026-04-30 访问

## why read today

你这轮转型前几天已经补了 `uv`、`Structured Outputs`、`MCP`、`Evals/Tracing`。如果继续只追 Agent 概念，很容易出现一个问题：脑子里全是协议、工具和工作流，但真正开始写 Python 项目时，仓库根目录该放什么、依赖写在哪里、测试怎么组织、怎么避免“本地能跑/装包后坏掉”，反而没有稳定习惯。

对做了 10 年 Java / Spring / IoT 集成 / ToB 交付的人来说，这一层很关键，因为你并不缺“系统边界意识”，缺的是把这种工程习惯迁移到 Python 世界的默认承载方式。`pyproject.toml` 就像 Python 项目的统一工程入口，`pytest` 推荐的 `src + tests` 布局则像把“可安装产物”和“源码目录”明确分开。Agent 项目如果没有这层地基，后面 MCP server、tool wrappers、eval harness、CLI 入口都会越堆越乱。

## original-text translation

Python Packaging 官方文档先把一件事讲得很明确：`pyproject.toml` 不只是给打包用的文件，它也是 lint、类型检查、构建等工具的共同配置入口。这个文件里最关键的三块分别是 `[build-system]`、`[project]` 和 `[tool]`。其中 `[build-system]` 用来声明你使用什么构建后端以及构建时需要哪些依赖；`[project]` 用来写项目元数据和依赖；`[tool]` 留给具体工具各自扩展。

官方还特别强调：无论你使用哪种构建后端，`[build-system]` 都应该存在。对新项目，优先使用 `[project]` 这套标准写法，而不是继续把核心配置散落在 `setup.py` 或旧格式里。也就是说，Python 新工程不应该再把“项目是什么、依赖是什么、命令入口是什么”分散到多个旧式文件中。

`pytest` 官方文档则从测试集成角度补上另一半地基。它建议开发时使用虚拟环境，用 `pip install -e .` 这样的可编辑安装来运行项目和测试，并推荐在仓库根目录放 `pyproject.toml`。在目录布局上，`pytest` 明确给出一种对新项目很友好的方式：使用 `src/` 存放应用代码，使用 `tests/` 放测试，并尽量采用 `importlib` 的导入模式。这样做的重点不是形式整齐，而是减少 `sys.path` 污染、避免测试误打误撞引用本地源码、让你更接近“安装后真实会怎样运行”的状态。

## Chinese deep summary

今天这两份官方文档，解决的其实是同一个问题：你要把 Agent 项目当成“一个正式 Python 软件包”，而不是“一个能跑的脚本目录”。

这对很多刚转 Python 的后端工程师特别重要。因为在 Java / Spring 世界里，你已经默认接受了这些工程事实：

1. 一个项目要有统一入口。
2. 依赖、构建、测试配置不能乱散。
3. 本地开发态和交付产物之间要尽量一致。
4. 测试不能偷偷依赖 IDE 或当前目录碰巧可用。

但 Python 新手最容易掉进反模式：根目录堆一堆 `.py`，再加几个 `requirements.txt`、临时脚本和 notebook，本地 `python main.py` 能跑就算开始了。做普通小工具也许问题不大，但做 Agent 工程，这种松散结构会很快放大成系统性成本。

先看 `pyproject.toml`。它的价值，不是“又一个配置文件”，而是把 Python 项目里过去分散的工程定义，尽量收敛到统一入口。你可以把它理解成：

1. `[build-system]`：告诉外部世界“这个项目如何被构建”。
2. `[project]`：告诉外部世界“这个项目是谁、依赖什么、支持什么 Python 版本、暴露什么命令入口”。
3. `[tool]`：告诉具体工具“它们各自的配置放哪儿”。

这和你熟悉的 `pom.xml` / Gradle build file 很像。它不是业务代码，但它决定了工程能不能被别人稳定地安装、测试、运行、发布。Agent 项目尤其需要这个统一入口，因为你后面大概率会同时挂上：

1. Web 服务入口，比如 FastAPI。
2. CLI 调试入口，比如跑 eval、导入样本、回放 trace。
3. 工具配置，比如 `pytest`、格式化、lint、类型检查。
4. 未来可能还会加 MCP server、worker、批处理脚本。

如果这些入口和依赖声明没有统一落点，仓库会很快变成“只有作者自己知道怎么启动”的状态。ToB 交付经验告诉你，这种项目一换机器、一换人、一进 CI 就开始出隐性问题。

再看 `pytest` 推荐的 `src + tests` 布局。很多人第一次看会觉得它只是目录风格偏好，但它背后其实是在防止一个很具体的 Python 坑：当前目录会影响导入行为。也就是说，如果你的代码直接放在仓库根目录，测试很可能不是在验证“安装后的包”，而是在验证“当前目录刚好能 import 到的源码”。这会让你产生一种危险错觉：测试全绿，但一旦打包、部署、换入口方式，问题才暴露。

`src` 布局的核心价值是，把“源码目录”和“仓库根目录”人为拉开一点距离，让导入路径更接近真实安装后的状态。再配合 `pip install -e .`，你本地开发时既能快速迭代，又更接近交付态。对 Agent 项目来说，这很有现实意义：

1. 你会有 prompt 模块、tool 定义、agent runtime、eval cases、配置文件。
2. 这些模块之间经常需要被 CLI、测试、服务入口、脚本重复引用。
3. 一旦导入边界模糊，后面拆模块、上 CI、做容器化时都会增加不必要的定位成本。

`pytest` 推荐 `importlib` 导入模式，也是在同一条线上。它的目标是减少测试运行时偷偷修改 `sys.path` 带来的惊喜。你可以把它理解成：尽量让测试环境少依赖“隐式路径魔法”，多依赖明确安装和明确导入。对 Java 背景的人，这几乎就是“类路径别靠运气”的 Python 版本。

还有一个很值得你记住的点：`pytest` 官方直接不推荐 `python setup.py test` 这一类旧方式，说明 Python 生态已经把项目测试从“setuptools 附属命令”转向“标准安装 + 标准测试工具”的组合。这个方向和 Agent 工程很契合，因为你后面会越来越依赖：

1. `pip install -e .` 或类似开发安装方式。
2. 独立的测试命令。
3. 独立的 eval 命令。
4. 独立的运行入口。

把今天的材料翻成一句更适合你当前阶段的话：

在 Agent 工程里，`pyproject.toml` 不是装饰文件，`src + tests` 也不是目录洁癖；它们是在提前解决“项目如何被安装、导入、测试、运行”这四个会越来越痛的问题。

## 3 key takeaways

1. `pyproject.toml` 是 Python 新项目的统一工程入口，至少要把构建系统、项目元数据和工具配置放到可被标准工具理解的位置。
2. `pytest` 推荐的 `src/ + tests/ + pip install -e .` 组合，目的是让本地开发更接近真实安装后的运行状态，减少导入路径导致的假阳性。
3. Agent 项目比普通脚本更需要清晰结构，因为后面会叠加 CLI、服务入口、工具定义、评测脚本和部署产物，早期不立规矩，后期会成倍返工。

## relation to Agent engineering

这篇内容和 Agent 工程的关系，不在“模型能力”层，而在“工程承载”层。

你后面如果用 Python 做 Agent，常见结构至少会包含：

1. `agents/` 或 `app/`：放 runtime、prompt 装配、tool 调用、状态流转。
2. `tests/`：放单测、集成测试、mock tool 场景。
3. `evals/` 或 `scripts/`：放批量评测、样本集、回归命令。
4. `pyproject.toml`：统一声明依赖、Python 版本、测试配置、CLI 入口。

这和你原来的 Java 工程观完全一致，只是载体变了。你不是在“补 Python 语法”，而是在学习 Python 世界里什么相当于 Maven/Gradle、什么相当于标准测试布局、什么相当于可交付工程的最小骨架。把这层补牢，后面学 FastAPI、MCP SDK、LangGraph、OpenAI Agents SDK 时，代码才有稳定落点。

## a small action for tonight

花 20 分钟，不追求写业务，只做一个最小骨架练习：

1. 画出你理想中的 Agent 项目目录：`pyproject.toml`、`src/your_agent/`、`tests/`、`evals/`。
2. 写一版最小 `pyproject.toml` 草稿，只包含 `[build-system]`、`[project]`、一个测试相关 `tool` 配置。
3. 问自己一句：如果明天把这个项目交给同事，他只看仓库根目录，能不能知道“怎么安装、怎么跑测试、怎么启动一个最小入口”？

## 原文关键段落翻译（人工翻译，放在文末）

1. Packaging 官方文档强调：`pyproject.toml` 是给打包工具以及 lint、类型检查等其他工具共同使用的配置文件。它不是单一工具私有格式，而是 Python 工程的公共入口。
2. 文档明确指出：`[build-system]` 这一节应该始终存在，因为它定义了你使用什么构建工具。对新项目，应优先采用 `[project]` 这套标准元数据写法。
3. pytest 官方文档建议：开发时使用 `venv` 和 `pip`，并通过 `pip install -e .` 进行可编辑安装，这样代码修改后可以持续重跑测试。
4. pytest 官方文档推荐新项目采用 `src` 布局，并建议使用 `importlib` 导入模式，因为这样能减少路径处理带来的常见陷阱。
5. pytest 还明确不推荐 `python setup.py test` 这类旧方式，说明现代 Python 测试流程应建立在标准安装与独立测试工具之上。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://packaging.python.org/en/latest/guides/writing-pyproject-toml/
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.pytest.org/en/latest/explanation/goodpractices.html
