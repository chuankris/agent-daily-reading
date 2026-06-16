# title

2026-06-16 为什么现在该把 Agent eval 落到 `pytest` 参数化回归集

## original source

- 标题：How to parametrize fixtures and test functions
- 链接：https://docs.pytest.org/en/stable/how-to/parametrize.html
- 来源类型：pytest 官方文档
- 访问时间：2026-06-16

- 标题：Good Integration Practices
- 链接：https://docs.pytest.org/en/stable/explanation/goodpractices.html
- 来源类型：pytest 官方文档
- 访问时间：2026-06-16

## why read today

你前两天刚补了 `behavior-driven evals`、`trace -> dataset -> improvement loop`，这套认知已经把“怎么定义 Agent 好坏”讲明白了。但如果停在这里，仍然容易卡在一个很典型的转型断层上：

**知道要做回归，不等于手里已经有一套能稳定重跑的回归机制。**

这正是今天这两篇 `pytest` 官方文档值得读的原因。它们讲的不是“AI 专属技巧”，而是把你熟悉的 ToB 验收思维，落成 Python 里最便宜、最可维护、最容易持续积累的执行载体。

对你这种做了 10 年 Java / Spring / IoT 集成、习惯按接口合同、验收口径、回归样例来控质量的人来说，`pytest` 参数化最有价值的地方不是语法糖，而是它能把：

- 一批真实用户输入
- 一组行为检查点
- 若干已知失败样例
- 每次修复后的重跑动作

压成一份很轻的、可以持续增量维护的最小回归集。

而 `Good Integration Practices` 这篇文档补的，则是另一个经常被低估的问题：**回归集写出来了，是否真的在测你以为自己在测的那份代码？**  
对 Agent 项目来说，这一点尤其关键。你后面会同时碰到本地代码、editable install、工具模块、MCP 封装层、测试夹具、临时数据目录。如果测试导入路径混乱，eval 结果就会掺进“环境偶然性”，最后既不稳定，也不可信。

所以今天这篇不是在换主题，而是在把前天的 eval 视角继续往前推一步：从“知道该怎么评”推进到“能把评估样例以 Python 工程方式持续跑起来”。

## original-text translation

`How to parametrize fixtures and test functions` 这篇文档首先把参数化说得很朴素：`pytest` 允许你在多个层次做参数化，包括 fixture、本身测试函数，以及更动态的 `pytest_generate_tests`。其中最常用的是 `@pytest.mark.parametrize`，它允许你把一组输入/期望值成批绑定到同一个测试函数上，让同一段断言逻辑按多组样例反复执行。文档强调，参数值会“原样传入”测试，不会自动拷贝；如果你把列表、字典这种可变对象直接作为参数传进去，并在测试里修改它，这个修改会污染后续样例。

文档还给了几个对回归集非常实用的能力。第一，单个参数样例可以单独打标记，比如把已知问题用 `pytest.param(..., marks=pytest.mark.xfail)` 标成“预期失败”，这样团队就能保留真实坏样例，而不是先把它删掉装作没看见。第二，多个 `parametrize` 可以叠加，形成组合矩阵；这很适合 Agent 场景里把“用户输入类型”“工具返回状态”“是否需要澄清”这类维度拆开，再自动组合出覆盖面更大的测试样例。第三，如果样例来自命令行或外部数据源，可以在 `pytest_generate_tests` 里动态生成参数；这给后面把 trace 抽成 dataset、再把 dataset 喂回回归测试留了接口。

`Good Integration Practices` 这篇文档讲的是另一个层面：测试工程本身要怎么摆，才能减少路径和导入带来的意外。文档建议新项目优先用 `src` layout，并推荐在新项目中使用 `--import-mode=importlib`。核心原因是默认的 `prepend` 模式会改写 `sys.path`，让测试文件以更“神奇”的方式被导入；项目一大，测试文件重名、导入到本地源码而不是安装后的包、或者目录结构影响测试行为，这些坑都会冒出来。相对地，`importlib` 模式不会为了导入测试模块去改动 `sys.path`，行为更不容易让人意外。

把这两篇连起来看，意思其实很明确：**参数化解决的是“怎样把一批行为样例压成可执行回归集”，集成实践解决的是“这套回归集是否在一个可重复、可解释的 Python 工程环境里运行”。** 对 Agent engineering 来说，前者让 eval 落地，后者让 eval 结果可信。

## Chinese deep summary

如果把今天这两篇材料压成一句话，就是：

**Agent eval 真正落地，不是多看几个评分面板，而是把真实失败样例沉淀成 `pytest` 参数化回归集，并保证它测的是正确的代码边界。**

第一层，你要把“评估思维”翻译成“样例矩阵思维”。

前天那篇行为型 eval 解决的是定义问题：不是只看最后答案像不像，而是看中间行为是否符合要求。今天 `pytest` 参数化这篇补的是执行问题：定义完以后，你拿什么来批量重放这些行为样例？

答案不是先上很重的平台，而是先把最小回归集做出来。  
对你来说，这很像以前做系统集成时的验收样例表。不同的是，现在这张表不是 Excel，而是一组参数化测试。每个样例可以包含：

- 用户输入
- 期望调用的工具或禁止调用的工具
- 是否必须先澄清
- 结构化输出里哪些字段必须存在
- 这条样例当前是通过、部分通过，还是已知失败

一旦你用 `@pytest.mark.parametrize` 把它们压进同一个测试函数，回归集就从“脑子里的经验”变成“代码里的资产”。

第二层，`xfail` 对 Agent 项目特别有价值，因为它允许你保留“还没修好的真实问题”。

很多人做回归时有个坏习惯：一旦某条样例老失败，就先删掉，或者挪到别处，等有空再说。结果数据集越来越“干净”，系统却没有更强。`pytest.param(..., marks=pytest.mark.xfail)` 给你的其实是一种更成熟的工程姿势：

- 失败样例继续留在主回归集中
- 团队明确知道它现在为什么不通过
- 真修好那天，它会从“预期失败”变成一个可以被摘掉标记的通过样例

这和你以前做客户交付时保留已知问题清单很像，只不过这里不是文档管理，而是测试执行层面的显式状态。

第三层，组合参数化很适合 Agent，不只是适合普通函数测试。

Agent 的坏，不经常只坏在一个维度。它可能在“模糊输入 + 外部接口超时 + 需要人工审批”这个组合下出问题，也可能在“结构化输出字段缺失 + tool retry 后仍继续瞎编”这种组合下出问题。  
叠加多个 `parametrize` 的价值，就是把这些维度从“手写 N 条大而全场景”变成“几个关键维度自动组合”。这样样例增长更可控，覆盖面却更高。

这对正在补 Python 的你尤其重要，因为它会把“写测试”的门槛降得很低。你不需要先学会一整套复杂框架，先把几个维度和断言写清楚，就已经是在做很像样的 Agent regression engineering 了。

第四层，动态参数化是把 trace / dataset 接进 Python 回归的桥。

如果你后面真的开始收集 trace、人工反馈、线上失败样例，那么这些数据最终总要找一个地方变成“可重跑的检查”。`pytest_generate_tests` 给出的不是花哨技巧，而是一个很实用的钩子：样例不必永远手写死在代码里，它们可以从命令行、JSON、导出的 trace 样本、甚至某个固定目录里的案例文件生成。

这意味着你未来可以很自然地走到这样一条路：

1. 线上 trace 发现失败
2. 抽出最小复现输入
3. 变成一个参数样例
4. 放入本地 `pytest` 回归集
5. 每次改 prompt、tool schema、routing、guardrail 时重跑

这条链一旦建起来，你的 eval 才真正从“看报告”进入“控回归”。

第五层，`Good Integration Practices` 解决的是一个更隐蔽但非常致命的问题：测试环境污染。

做 Agent 工程的人很容易把注意力都放在模型和工具上，却忽视测试到底 import 了哪份代码。文档建议新项目优先 `src` layout，并推荐 `--import-mode=importlib`，本质上就是在帮你减少这类假阳性和假阴性：

- 明明以为测的是安装后的包，实际测的是仓库里的本地源码
- 不同目录下同名测试文件互相干扰
- `sys.path` 被悄悄改写后，测试结果受目录结构影响
- 本地跑得过，换个 CI 或换个虚拟环境就不一致

对 Agent 项目而言，这种污染特别危险，因为你本来就已经在处理模型不确定性、外部工具不稳定性、上下文依赖性。如果连测试导入边界都不稳定，最后你根本分不清问题到底出在模型、工具合同、业务代码，还是测试装配方式。

第六层，这正好贴合你的迁移优势。

你不是从零学习“怎么做软件质量”，你是在把原来 Java / Spring / IoT 集成里的那套验收与回归能力，搬到 Python/Agent 世界。今天最值得迁移过来的，不是某个具体 API 名字，而是这三条手感：

- 每个真实问题都尽量沉淀为一个可重跑样例
- 每次修改都尽量跑一组最小回归集
- 回归结果必须建立在可重复的测试装配边界上

一旦这三条建立起来，你后面学 Datasets、Agent SDK、MCP tool eval、甚至多 Agent workflow，都会更稳。因为你不再只是“会搭 Agent”，而是在建立一个能持续防回归的 Agent 工程底座。

## 3 key takeaways

1. `pytest.mark.parametrize` 的核心价值不是省代码，而是把一批真实 Agent 行为样例压成可重跑、可增量维护的最小回归集。
2. `pytest.param(..., marks=pytest.mark.xfail)` 允许你把“已知失败但必须保留”的真实问题留在主回归集中，这比删样例更符合工程现实。
3. `src` layout 和 `--import-mode=importlib` 这类测试集成实践，解决的是“回归到底测到了哪份代码”这个根问题；没有这层稳定性，eval 结果就不可信。

## relation to Agent engineering

这和 Agent engineering 的关系非常直接。

第一，它把你前几天学的 eval、trace、dataset 认知，接到了一个能每天执行的 Python 工程载体上。以后你不必把“评估”理解成单独平台，也可以把它理解成一组围绕 Agent 行为的参数化测试。

第二，它会倒逼你把 Agent 的中间行为设计得更可断言。比如工具调用名称、结构化输出字段、错误分类、审批分支、澄清分支，只要你想测，就必须先把这些边界设计得更清楚。这会反过来提高 tool contract、routing、guardrail 的质量。

第三，它特别适合你的背景。你原来最擅长的不是炫技式 demo，而是把跨系统能力做成可验收、可回归、可交付的东西。把 `pytest` 参数化用于 Agent 回归，本质上是在把这套长处继续发挥出来，只是语言从 Java 多了一部分 Python，系统边界从 REST/MQ 扩展到了模型与工具。

## a small action for tonight

今晚做一个 40 分钟以内的小动作，不求把平台接全，只求把“行为型 eval -> pytest 回归集”走通一次：

1. 选你最近一个最熟的 Agent 场景，写出 6 条真实输入，其中至少 2 条是失败或边界样例。
2. 给每条样例补 3 个行为检查点，例如“是否先澄清”“是否调用正确工具”“是否输出了必须字段”。
3. 用 `@pytest.mark.parametrize` 把这 6 条样例压进一个测试函数里，不要先追求复杂框架。
4. 把其中 1 条你确认暂时修不掉的问题，显式标成 `xfail`，练习“保留坏样例而不是删除坏样例”。
5. 检查项目测试配置里是否仍依赖默认导入行为；如果是，记录一条待办：切到 `--import-mode=importlib`，并确认回归测到的是你真正想测的代码路径。

## 原文关键段落翻译（人工翻译，放在文末）

1. `pytest` 可以在多个层次支持参数化：可以参数化 fixture，可以用 `@pytest.mark.parametrize` 给测试函数或测试类批量提供参数，也可以用 `pytest_generate_tests` 自定义更动态的参数化方案。
2. `@pytest.mark.parametrize` 会让同一个测试函数按多组参数重复执行；传入的参数值不会被自动拷贝，所以如果样例里包含可变对象并在测试中被修改，这种修改会影响后续样例。
3. 你可以对单个参数样例单独打标记，例如把某一条样例标成 `xfail`；这样它仍然保留在测试集中，但会被明确视为“预期失败”。
4. 多个 `parametrize` 装饰器可以叠加，形成参数组合；这意味着你可以把多个维度拆开定义，再让 `pytest` 自动展开成完整测试矩阵。
5. 对新项目，`pytest` 官方建议优先使用 `src layout`，并推荐使用 `--import-mode=importlib`；因为默认的 `prepend` 模式会改动 `sys.path`，更容易带来导入行为上的意外。
6. `importlib` 模式的一个重要好处是：为了导入测试模块，`pytest` 不需要修改 `sys.path`，因此测试导入边界更清晰，也更不容易误测到本地目录里的代码。

## 原文中文翻译链接（机器翻译）

- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.pytest.org/en/stable/how-to/parametrize.html
- https://translate.google.com/translate?sl=auto&tl=zh-CN&u=https://docs.pytest.org/en/stable/explanation/goodpractices.html
