# LLM-06｜场景｜LLM 在运维与容灾中的落地：SRE 实战地图

> 🎯 **这篇文档解决什么问题**：大家都在说“用大模型做运维”，但真到值班、复盘、容灾治理的现场，常见的问题是：哪些活适合交给 LLM？它能自己动手到什么程度？出了错谁兜底？怎么证明它真的有用？本文把业界 2023–2026 年的论文、基准和工业实践，整理成一张按**事故生命周期**展开的地图。每个场景都回答四件事：**价值在哪**、**成熟度如何**、**风险是什么**、**该给几级自主权**。最后落到参考架构、安全护栏、评测方法和分阶段路线图。
>
> 使用方式：先读第 1、2 章判断哪些场景值得做，第 3 章定自主等级；第 4、5 章逐个场景展开，搭系统的读第 6、7 章，第 8 章讲怎么证明有用，第 9 章给路线图。读之前了解 Prompt、RAG、Agent 和工具调用这些基本概念即可。文中基准分数核对于 2026 年 10 月 11 日。

## 导读 · 先给结论

- **现在的 Agent 还远不能独立处理真实故障**。在 OpenRCA 基准上，表现最好的模型配上专门设计的 Agent，完全答对的比例也只有约 11%[[1]](https://proceedings.iclr.cc/paper_files/paper/2025/hash/d29b8d53678015079e1d245c023e49d2-Abstract-Conference.html)；IBM 的 ITBench 论文（2025）中，SRE 场景的解决率约 11%–14%[[2]](https://arxiv.org/abs/2502.05352)；2026 年 IBM 与 Artificial Analysis 推出的 ITBench-AA 只考根因诊断，最好的模型（Claude Opus 4.7）也只有 47%[[27]](https://huggingface.co/blog/ibm-research/itbench-aa)。
- **把专家经验写成流程（SOP）再交给 Agent，效果提升最大**。同一模型，按 SOP 约束后根因准确率从 35.5% 提到 64.0%[[3]](https://arxiv.org/abs/2502.08224)。
- **数值判断交给传统算法**。时序异常检测上，专门的深度学习模型比 LLM 好约 30%[[4]](https://arxiv.org/abs/2405.14755)。
- **先只读、后可写；先建议、后执行**。高风险动作始终留在人手里，自动执行只开放给白名单内、幂等、可回滚的动作。

一次线上事故从“还没发生”到“彻底关闭”，可以切成三段。LLM 在每一段里能接手的工作性质不同：

- <b>事前（预防）</b>：变更风险评审、预案生成与检查、单点与未多活识别、演练设计。特点是**不赶时间、可以慢慢查证**，最适合先落地。
- <b>事中（响应）</b>：告警聚合、事故摘要、根因分析（RCA）、生成查询语句、给出止损建议、对外播报。特点是**时间紧、信息乱、动作有风险**，LLM 的价值最大，出错的代价也最大。
- <b>事后（改进）</b>：复盘文档、改进项跟踪、知识沉淀成 Runbook。特点是**文本密集、格式固定**，是 LLM 的舒适区。

![LLM 在事故生命周期中的位置](assets/06-llm-for-sre/fig-01.jpg)

贯穿全文的一条主线是：<b>LLM 负责“读懂、编排、写出来”，确定性系统负责“算数、执行、兜底”</b>。几乎所有成功案例都遵循这条分工，几乎所有翻车案例都违反了它。

| 章节 | 回答的问题 | 图 |
|---|---|---|
| 第 1 章 契合点与边界 | 运维数据里哪些适合 LLM，哪些必须交给传统算法 | A2 |
| 第 2 章 场景地图 | 十几个场景的价值、成熟度、风险排序 | A3 |
| 第 3 章 自主等级 | 从“只给建议”到“自动执行”，每一级的准入条件 | A4 |
| 第 4 章 关键场景详解（上） | 告警降噪、日志分析、根因分析 Agent 与公开基准成绩 | A5–A8 |
| 第 5 章 关键场景详解（下） | 自然语言查数、变更风险、复盘生成、容灾治理 | A9 |
| 第 6 章 参考架构 | 数据、知识、工具、Agent、护栏、评测六层怎么搭 | A10 |
| 第 7 章 安全护栏 | 日志里的提示注入、越权、幻觉命令怎么防 | A11 |
| 第 8 章 评测 | 怎么证明它有用：回放、影子模式、指标与统计 | A12 |
| 第 9 章 路线图与反模式 | 分四个阶段推进，每阶段的退出条件；常见的坑 | A13 |

📎 与系列其他文档的分工：LLM 是什么、怎么工作见「LLM 基础」，读懂公式需要的数学见「LLM 数学补课」。运维 Agent 的基准分数和生产案例见「AIOps Agent 落地」，Agent 的通用工程做法见「Agent 工程实战」。本文只讲 LLM 在事故的各个阶段能做什么、自主权给到几级。

## 第 1 章 · 契合点与边界：LLM 擅长什么，不擅长什么

运维数据有八九种，形态完全不同。LLM 本质上是一个**文本序列模型**：它擅长理解和生成自然语言、代码、半结构化文本，擅长在多段信息之间建立联系；它不擅长精确计算、长数值序列里的模式判断，也不保证同样输入每次给出同样输出。把这两点和数据形态对上，就能得到第一张分工表。

| 数据类型 | 形态 | 适合交给 LLM | 应交给确定性系统 |
|---|---|---|---|
| 日志（Logs） | 半结构化文本，量极大，模板重复 | 解释陌生日志的含义、归纳异常模式、从报错里提取关键实体 | 模板提取（Drain 等）、计数、按模板聚合、全量扫描 |
| 指标（Metrics） | 数值时序 | 解释“这条曲线在说明什么”、选择该看哪些指标 | 异常检测、阈值判断、同比环比、相关性计算 |
| 链路（Traces） | 调用图 + 时延数值 | 读懂一条异常链路的上下游关系、写出人能看懂的解释 | 拓扑构建、关键路径计算、耗时统计 |
| 告警与事件 | 短文本 + 时间戳 + 标签 | 判断两条告警是否同一原因、生成事故标题与摘要 | 去重、按时间窗与拓扑做初步关联 |
| 变更记录 | 结构化字段 + 代码 diff + 描述 | 读 diff 判断影响面、对照预案检查遗漏项 | 变更与故障的时间对齐、灰度指标比对 |
| 工单、聊天记录 | 长篇自然语言 | 摘要、时间线还原、行动项抽取 | 按事故 ID 归档、权限控制 |
| Runbook、SOP、复盘 | 长文档、经验知识 | 检索相关段落、把文档转成可执行步骤 | 版本管理、过期检测 |
| 拓扑、CMDB、容灾标准 | 结构化、权威 | 把问题翻译成查询参数 | **唯一事实来源**，LLM 只读不写、不猜 |

![分工原则：LLM 读懂与编排，确定性系统算数与执行](assets/06-llm-for-sre/fig-02.jpg)

### 1.1 为什么数值判断不交给 LLM

直觉上，“让 GPT 看一眼曲线判断是否异常”很方便。但 MIT 团队的系统评测给出了清楚的答案[[4]](https://arxiv.org/abs/2405.14755)：他们把时序转成文本交给 LLM，在 11 个数据集、492 条时序上测试。直接问 LLM“哪里异常”效果很差；让 LLM 做预测、再按预测残差判断异常，效果明显更好，但仍比当前最好的专用深度学习模型（AER）低约 30%。

原因不难理解：数字被切成 token 之后，模型对“0.98 和 1.02 的差距”“过去 1440 个点的周期”并没有天然的感知；长序列还会挤占上下文。专门的时序基础模型（如 Chronos[[5]](https://arxiv.org/abs/2403.07815)、TimesFM[[6]](https://arxiv.org/abs/2310.10688)）是另一条路：它们把时序当成原生输入来训练，而不是当文本处理。工程上更稳的做法是：**检测交给专用模型或规则，LLM 读检测结果并解释**。

### 1.2 为什么事实不交给 LLM 记

LLM 对“某服务在哪几个机房”“某简写指哪个机房”这类问题，会给出看起来很合理的答案，但它的答案来自训练数据里的统计规律，不来自你的 CMDB。运维场景里，这类“看似合理的错”比明显的错更危险，因为它会顺着后续步骤一路传下去。团队的容灾 Agent 手册把这条写成了红线：机房简写只能通过白皮书拓扑清单解析，**碰巧猜对也算违规**（详见 5.4 节）。

> 📌 **三条判断规则**，拿到一个想用 LLM 做的运维任务时先过一遍：
>
> 1. <b>输出能不能被确定性地校验？</b>能（例如生成的 SQL 可以试跑、生成的摘要可以和原文对照），就适合；不能，就必须有人审。
> 2. <b>输入里有没有大量数值要比较？</b>有，就先用传统算法把数值变成结论（“p99 时延在 14:02 突增 3 倍”），再交给 LLM。
> 3. <b>输出会不会直接改变线上状态？</b>会，就落到第 3 章的自主等级里去讨论，默认只给建议。

## 第 2 章 · 场景地图：先做哪些，后做哪些

下表按事故生命周期列出常见场景。“成熟度”指业界是否已有可复现的论文或公开的生产部署；“建议自主等级”对应第 3 章的 L0–L4。排序原则是：**价值高、成熟度高、风险低的先做**。

| 阶段 | 场景 | 价值 | 成熟度 | 主要风险 | 建议等级 | 代表工作 |
|---|---|---|---|---|---|---|
| 事前 | 变更风险评审 | 读 diff 与变更单，对照历史事故提示风险点 | 中 | 漏报带来虚假安全感 | L1 | 灰度分析系统 Gandalf[[7]](https://www.usenix.org/conference/nsdi20/presentation/li)（非 LLM，可作为信号源） |
| 事前 | 预案生成与检查 | 按模板补全预案，检查遗漏的回滚步骤 | 中 | 预案“看起来完整”但不可执行 | L1 | Nissist 把排障手册转成可执行节点[[8]](https://arxiv.org/abs/2402.17531) |
| 事前 | 单点与未多活识别 | 把容灾看板数据解读成可行动的清单 | 中 | 口径混用、数据滞后、把“未多活”说成“单点” | L1 | 团队容灾 Agent（5.4 节） |
| 事前 | 演练设计 | 按架构与历史故障生成故障注入清单 | 低 | 爆炸半径估计错误 | L1 | AIOpsLab 的故障注入框架[[9]](https://arxiv.org/abs/2501.06706) |
| 事中 | 告警聚合与降噪 | 告警风暴里把几百条告警归成几个事件 | 高 | 错误合并导致漏掉独立故障 | L2 | COLA[[10]](https://arxiv.org/abs/2403.06485)、iPACK[[11]](https://arxiv.org/abs/2302.09520) |
| 事中 | 事故摘要与播报 | 给新加入的人 30 秒讲清现状 | 高 | 摘要遗漏关键事实 | L2 | RCACopilot 的事故摘要步骤[[12]](https://arxiv.org/abs/2305.15778) |
| 事中 | 生成查询语句 | 自然语言转 SQL / PromQL / KQL | 高 | 查询语义错但能跑通 | L2 | Xpert[[13]](https://arxiv.org/abs/2312.11988) |
| 事中 | 根因分析 | 缩小排查范围，给出带证据的候选根因 | 中 | 自信地给出错误根因 | L1 | RCACopilot[[12]](https://arxiv.org/abs/2305.15778)、RCAgent[[14]](https://arxiv.org/abs/2310.16340)、Flow-of-Action[[3]](https://arxiv.org/abs/2502.08224) |
| 事中 | 止损建议与执行 | 推荐切流、回滚、扩容等动作 | 低 | 错误动作扩大故障 | L1（白名单动作 L3） | ITBench 的 SRE 任务[[2]](https://arxiv.org/abs/2502.05352) |
| 事后 | 复盘文档生成 | 从群聊、时间线、指标整理成结构化复盘 | 高 | 编造原因或责任人 | L2 | Google SRE 复盘文化[[15]](https://sre.google/sre-book/postmortem-culture/)、团队 dr-postmortem skill |
| 事后 | 改进项跟踪 | 抽取行动项、提醒逾期、合并重复项 | 高 | 低 | L3 | — |
| 事后 | 知识沉淀 | 把复盘与排障经验转成 Runbook / SOP | 中 | 过期知识被当成真理 | L2 | Nissist[[8]](https://arxiv.org/abs/2402.17531) |

![场景排序：价值高、成熟度高的先做](assets/06-llm-for-sre/fig-03.jpg)

> 📊 **怎么读这张表**：右上角（价值高、成熟度高）是告警聚合、事故摘要、查询生成、复盘生成，这四个场景的共同点是**输出给人看，且能被人快速核对**。根因分析价值最高，但成熟度只有“中”，适合先做“缩小范围 + 给证据”，不做“下结论”。止损执行价值虽高，但目前公开基准上的成功率很低，应该最后做，且只开放白名单动作。

## 第 3 章 · 自主等级：LLM 能自己动手到什么程度

“要不要让 Agent 自动执行”要分级回答，没法只答是或否。借鉴自动驾驶的分级思路，可以把运维 Agent 的自主权分成五级。**同一个 Agent 可以在不同动作上处于不同等级**：它可以 L3 自动发事故摘要，同时在切流这件事上只能 L1 给建议。

| 等级 | 名称 | Agent 做什么 | 人做什么 | 典型动作 |
|---|---|---|---|---|
| L0 | 无辅助 | 不参与 | 全部 | — |
| L1 | 只读建议 | 只读取数据，给出分析和建议，附证据 | 判断并执行 | 根因候选、止损建议、风险提示 |
| L2 | 起草待批 | 生成可直接执行的产物（命令、查询、文档、变更单），等人确认 | 审核、点确认 | 生成查询、起草复盘、起草播报 |
| L3 | 白名单自动 | 在白名单内自动执行，执行后通知 | 事后抽查，可随时叫停 | 发摘要、建群拉人、重启单个无状态实例 |
| L4 | 完全自主 | 自主决定并执行任意动作 | 不介入 | **当前不建议任何生产场景使用** |

![自主权分级：要“挣”来，不是“给”出去](assets/06-llm-for-sre/fig-04.jpg)

### 3.1 升到 L3 的五个准入条件

一个动作要从 L2 升到 L3（自动执行），必须同时满足下面五条。缺任何一条，就停在 L2：

1. **幂等**：重复执行两次和执行一次结果相同。Agent 会重试、会被中断，非幂等动作在重试时会放大影响。
2. **可回滚**：有明确、经过验证的回滚路径，且回滚本身也是自动化的。
3. **爆炸半径有上限**：单次动作的影响范围有硬限制，例如“一次最多重启 1 个实例”“一次最多影响 5% 流量”，由执行层强制，不靠 Prompt 约束。
4. **有前置检查**：执行前由确定性代码校验前提条件（目标是否健康、是否在变更窗口、是否已有人在处理）。
5. **有历史证据**：这个动作在 L2 阶段已经被人批准过足够多次，且人工修改率很低（见第 8 章的“采纳率”）。

> ❗ <b>切流、回滚发布、删除数据、修改容灾配置，这类动作不进 L3。</b>原因在于这些动作的后果不可逆或影响面不可控，跟 LLM 聪不聪明无关。公开基准上，Agent 在真实 SRE 任务中的严格解决率只有一到两成[[2]](https://arxiv.org/abs/2502.05352)，2026 年的新基准上只做诊断也不到一半[[27]](https://huggingface.co/blog/ibm-research/itbench-aa)，现阶段让它独立决定这类动作，风险远大于收益。

### 3.2 自主权要“挣”来，不是“给”出去

一个实用的做法是把等级和数据绑定：每个动作类型单独统计“建议被采纳率”和“采纳后的修改率”。连续一段时间（例如 3 个月、至少 50 次）采纳率高于 95%、且没有造成事故，才评审是否升级；任何一次由 Agent 动作引起的事故，该动作类型立即降回 L2。这样自主权的边界由证据决定，而不是由热情决定。

## 第 4 章 · 关键场景详解（上）：告警、日志、根因

📎 延伸阅读：时序、日志和 Trace 上的基础模型（容量预测、异常检测、日志解析）见 [LLM-27｜场景｜运维数据上的基础模型：时序、日志与 Trace](./27-ops-foundation-models.md)。

### 4.1 告警聚合与降噪

告警风暴是值班最常见的痛点：一个底层故障沿着依赖链往上传，几分钟内触发几百条告警。目标是把它们归成少数几个“事件”，每个事件对应一个独立原因。

纯规则（按时间窗、按标签）召回高但会误合并；纯 LLM 逐条两两比较成本太高、也不稳定。业界的成熟做法是**混合式**：先用统计方法挖掘告警之间的共现关系，把确定能合并和确定不能合并的处理掉；剩下模棱两可的少数几对，再交给 LLM 读告警内容和相关 SOP 做判断。COLA 就是这个思路，在一家大型云厂商的生产数据上 F1 达到 0.901–0.930[[10]](https://arxiv.org/abs/2403.06485)。更早的 iPACK 把告警解析成事件图，再与客户工单关联，用于合并重复工单，F1 为 0.871–0.935[[11]](https://arxiv.org/abs/2302.09520)。

![告警聚合：统计先筛，LLM 只判断拿不准的](assets/06-llm-for-sre/fig-05.jpg)

> ✅ **落地要点**：LLM 只处理“相关性挖掘拿不准”的那一小部分；合并决策要可撤销（值班同学能一键拆开）；指标上同时看“压缩比”和“误合并率”，只看压缩比会鼓励过度合并，把两个独立故障藏进一个事件里。

### 4.2 日志分析：模板优先，LLM 补语义

日志分析的第一步是**解析**：把 `Connection to 10.2.3.4:6379 timed out after 3000ms` 变成模板 `Connection to <*> timed out after <*>ms` 加参数。传统方法 Drain 用固定深度的解析树做在线解析，速度快，至今仍是工业界的基线[[16]](https://doi.org/10.1109/ICWS.2017.13)。它的弱点是对新格式、语义相近但写法不同的日志不够鲁棒。

LLM 的加入方式有两种：

- **直接让 LLM 解析和判断异常**。LogPrompt 通过自我提示、思维链、上下文示例等 Prompt 策略，在不使用领域训练数据的情况下，效果比用上千条日志训练的方法最多高 55.9%；6 位有 10 年以上经验的工程师给它生成的解释打了平均 4.42/5 分[[17]](https://arxiv.org/abs/2308.07610)。缺点是每条日志都调用一次 LLM，成本和时延扛不住生产流量。
- **LLM + 解析缓存**。LILAC 把 LLM 解析出的模板存进自适应缓存，新日志先查缓存，命中就直接用，只有没见过的日志才调用 LLM。在 Loghub-2.0（14 个数据集、平均每个 360 万行）上，它的模板准确率 F1 比此前最好的方法高 69.5%，而 LLM 调用次数下降几个数量级，速度和 Drain 相当[[18]](https://arxiv.org/abs/2310.01796)[[19]](https://arxiv.org/abs/2308.10828)。

![日志解析：高频走模板，长尾走 LLM](assets/06-llm-for-sre/fig-06.jpg)

生产上推荐第二种：**高频日志走模板，长尾和新出现的日志走 LLM**。事故中真正有用的，往往正是那些“以前没见过”的日志行。

### 4.3 根因分析 Agent

根因分析是 LLM 运维研究最集中的方向。按时间顺序看，方法从“一次性生成”走向“带工具的 Agent”，再走向“受 SOP 约束的 Agent”：

| 工作 | 来源 | 做法 | 关键结论 |
|---|---|---|---|
| Ahmed et al. | Microsoft，ICSE 2023 | 用 GPT-3.x 系列，根据事故标题和描述直接生成根因与止损建议；比较零样本、微调、多任务三种设置 | 4 万多起真实事故、1000 多个服务；微调显著好于零样本[[20]](https://arxiv.org/abs/2301.03797) |
| RCACopilot | Microsoft，EuroSys 2024 | 按告警类型匹配预先写好的诊断处理器自动收集信息，LLM 摘要后，用少样本思维链预测根因类别并给出解释 | GPT-4 Micro-F1 0.766；诊断信息收集模块已在 30 多个团队使用 4 年以上[[12]](https://arxiv.org/abs/2305.15778) |
| RCAgent | 阿里云，CIKM 2024 | ReAct 式 Agent，自由调用日志、代码分析工具；在动作轨迹层面做自洽性投票；使用内部部署模型以保护数据 | 全面优于 ReAct；已接入实时计算 Flink 平台的诊断流程[[14]](https://arxiv.org/abs/2310.16340) |
| D-Bot | 清华，VLDB 2024 | 从数据库诊断文档中抽取知识，树搜索探索多条诊断路径，多个 LLM 专家协作处理多根因 | 539 个异常；几分钟内给出诊断报告，DBA 通常需要数小时[[21]](https://arxiv.org/abs/2312.01454) |
| Flow-of-Action | WWW 2025 | 把 SRE 的排查经验写成 SOP，Agent 先匹配 SOP，没有就生成，再把 SOP 转成代码执行；辅助 Agent 负责去噪和判断何时停止 | 90 个注入故障上平均准确率 64.01%，ReAct 为 35.50%[[3]](https://arxiv.org/abs/2502.08224) |

![根因分析 Agent：给自由发挥加约束](assets/06-llm-for-sre/fig-07.jpg)

这条演进线的启示很直接：**模型自由发挥越多，越容易走偏**。ReAct 式 Agent 的主要失败模式是一步幻觉导致后续动作全部跑偏；有效的改进几乎都在给 Agent “加约束”：预定义的诊断处理器（RCACopilot）、轨迹投票（RCAgent）、SOP（Flow-of-Action）。对团队来说，最值钱的资产是**写清楚的排障 SOP 和可调用的诊断工具**，模型反而排在后面。

### 4.4 公开基准：现在的 Agent 到底有多强

论文里的数字大多来自作者自己的数据集，彼此不可比。2025 年开始出现了几个公开的、可复现的基准，结论高度一致：**离独立处理真实故障还很远**。

| 基准 | 来源 | 任务 | 代表成绩 |
|---|---|---|---|
| OpenRCA | Microsoft 等，ICLR 2025 | 3 个企业系统的 335 起故障、68 GB 遥测数据（日志、指标、链路），要求给出根因的时间、组件和原因 | （官方排行榜，截至 2025-01-23）Claude 3.5 配合专门设计的 RCA-agent，完全答对 11.34%；GPT-4o 8.96%；不用 Agent 时最好只有约 5%[[1]](https://proceedings.iclr.cc/paper_files/paper/2025/hash/d29b8d53678015079e1d245c023e49d2-Abstract-Conference.html) |
| ITBench | IBM，ICML 2025 | 真实环境中的 SRE、安全合规（CISO）、成本优化（FinOps）三类任务，102 个场景 | （ICML 2025 定稿版）SRE 场景解决率 11.4%，CISO 25.2%，FinOps 25.8%[[2]](https://arxiv.org/abs/2502.05352)；arXiv 初版为 94 个场景、SRE 13.8%。2026 年的 ITBench-AA 只测 SRE 根因诊断（59 个 Kubernetes 故障），最高 47%[[27]](https://huggingface.co/blog/ibm-research/itbench-aa) |
| AIOpsLab | Microsoft，MLSys 2025 | 自动部署微服务、注入故障、产生负载，Agent 通过统一接口完成检测、定位、根因、止损四类任务 | 提出“AgentOps”评测框架，可复现地对比不同 Agent[[9]](https://arxiv.org/abs/2501.06706) |

![公开基准：当前 Agent 的真实水位](assets/06-llm-for-sre/fig-08.jpg)

> ❗ **不要把这些数字横向比较**：OpenRCA 的 11% 和 Flow-of-Action 的 64% 不是一回事，前者是真实企业数据上的严格完全匹配，后者是单一演示系统上的注入故障。能确定的结论只有一个：在真实复杂度下，当前 Agent 是一个**能帮你缩小范围的助手**，不是一个能替你下结论的值班工程师。

## 第 5 章 · 关键场景详解（下）：查数、变更、复盘、容灾治理

📎 延伸阅读：自然语言查数（Text-to-SQL、PromQL / LogQL 生成）的方法、语义层与护栏，见 [LLM-39｜场景｜Text-to-SQL 与数据分析 Agent：从自然语言到可信查数](./39-text-to-sql.md)。

### 5.1 自然语言查数：从“会写查询”到“写对查询”

事故中，值班同学大量时间花在写查询上：SQL、PromQL、日志检索语句。Microsoft 的 Xpert 统计了一个大型云管理系统两年的事故与对应的 KQL 查询，发现查询高度依赖事故上下文，而且相似事故的查询有很强的模式。Xpert 先检索历史上相似的事故和当时用过的查询，再作为少样本示例交给 LLM 生成新查询，生成后校验语法，并把新的标准答案持续加入向量库。它还提出了一个专门评价查询质量的指标 Xcore，并已部署在生产环境[[13]](https://arxiv.org/abs/2312.11988)。

“能跑通”和“查对了”差得很远：一条漏了过滤条件的查询照样返回结果，只是数字是错的。生产上的四条做法：

1. **给 Schema，不让猜**：把表名、字段名、枚举值、常用过滤条件作为上下文提供，而不是让模型凭记忆写。
2. **只读账号 + 资源上限**：执行账号只有查询权限，查询自动加时间范围和行数上限。
3. **回显查询本身**：把生成的查询和结果一起展示给人，让人能一眼看出“过滤条件对不对”。
4. **回显数据新鲜度**：报数字时同时报出数据分区日期，避免把几天前的数据当成现在（见 5.4 节的 C4）。

### 5.2 变更风险评审

大量线上故障由变更引起。这里 LLM 和传统系统的分工很清楚：

- **传统系统**负责“变更后指标有没有变坏”。例如 Microsoft 的 Gandalf 在灰度过程中关联部署事件与故障信号，自动判断是否该阻断发布[[7]](https://www.usenix.org/conference/nsdi20/presentation/li)。这是数值和统计问题，不交给 LLM。
- **LLM**负责“变更前读懂改了什么”：读 diff 和变更描述，指出影响面（改了超时配置、改了重试逻辑、改了数据库 schema），检索历史上同类变更引发过的事故，对照预案检查是否缺少回滚步骤和观察指标。

输出形式应是**风险提示清单**，而不是“通过/不通过”的结论。LLM 漏报的代价是虚假的安全感，所以它只能增加评审者的视角，不能替代评审。

📎 评审之外，变更风险还能在合入前先打一次分，发布时再由灰度拦一道。Meta 的 Diff Risk Score 和灰度门禁的具体做法见第 56 篇 [LLM-56｜场景｜事件管理与变更 Agent：告警降噪、根因分析与发布门禁](./56-incident-change-agent.md) 第 8、9 章。

### 5.3 复盘文档生成

复盘是 LLM 最容易产生价值的场景：材料（群聊、时间线、告警、指标截图）都是现成的，格式是固定的，读者是人。Google SRE 的复盘文化强调**对事不对人**，关注系统和流程为什么允许错误发生，而不是谁犯了错[[15]](https://sre.google/sre-book/postmortem-culture/)。这一点也要写进 Prompt。

团队的 `dr-postmortem` skill 给出了容灾演练复盘的完整框架：9 章结构、问题按 P0/P1/P2 定级，每个问题必须填 9 个字段：现象、影响、根因、为何演练前没发现、临时处理、长期修复、Owner、DDL、验证标准。这个框架的价值在于它把“好复盘”变成了可以逐项检查的清单。

> ❗ **复盘生成有两条红线**。第一条是**不编造**，材料里没有的根因、Owner、DDL 一律标“未提供”，交给人补，绝不让模型“合理推测”出一个责任人。第二条是**每条结论带出处**，时间线的每个时间点、每个数字都能回链到原始消息或看板。

### 5.4 容灾治理：一个完整的团队案例

容灾治理问的典型问题是：“某业务域的 P0/P1 服务多活满足度是多少？差在哪？哪些是单点？”这类问题看起来是查数，实际上是一个**多源事实的合成**问题。团队的容灾编排 Agent 把它拆成四根相互独立的轴：

| 轴 | 含义 | 由谁提供 |
|---|---|---|
| 应是（标准） | 白皮书规定的阈值、公式、部署规格 | `dr-standard-reader`，指向容灾白皮书 |
| 实是（实测） | 线上看板测到的当前值 | `tiktok-dr-data-guide` → `aeolus-x` |
| 身份（归属） | 这个服务归哪个容灾域 / 业务域 | `bizdomain-resolver` |
| 合成 | 差距 Gap = 应是 − 实是，满足度 Sat = 实是 / 应是 | `dr-gap-resolver`，纯 Python 确定性计算 |

![容灾治理：数字来自工具，LLM 只做路由与表达](assets/06-llm-for-sre/fig-09.jpg)

这个设计正好体现了第 1 章的分工原则：LLM 负责理解问题、路由到正确的 skill、把结果写成人能读的结论；**每一个数字都来自确定性的工具，Gap 和 Sat 由不含 LLM 的代码计算**。在此基础上，手册设了四道“闸门”，任何一道没过就停下报错：

| 闸门 | 名称 | 要求 |
|---|---|---|
| C1 | 机房码接地 | 机房简写只能经白皮书拓扑清单解析，取数前回显“简写 → 清单条目 → 机房码 → region”的解析链；不许按字面猜，不许在代码里写死映射表 |
| C2 | 身份先行、口径隔离 | 涉及业务域时先跑身份解析并回显归属与成员集来源；容灾口径（dept）和业务域口径（domain）是两套独立分类，混用则结论作废 |
| C3 | 单点口径出处与对账 | “单点”定义必须引用白皮书的多活部署边界；每条单点定位到服务 × 业务域 × 调用方向 × 服务等级；输出前校验分项之和等于总数 |
| C4 | 实是新鲜度 | 报任何“当前”数字前，先回显数据日期、当前日期、滞后天数、本次使用的默认筛选；滞后超过阈值不当作当前值 |

手册里记录的几条教训，对所有运维 Agent 都适用：

- **碰巧对 ≠ 合规**。Agent 把一个机房简写按字面猜成了某个地名，结果碰巧对了。但这次碰巧对，下次换一个简写就会错，而且没人能发现。所以规则是：说不出出处就停下。
- <b>“未多活”不能说成“单点”</b>。看板的“未多活”名单是按“有没有覆盖指定的几个机房”判定的，实测某组件 25 个对象全部部署在 2~3 个机房，没有一个是字面意义的单点。把名单直接叫“单点列表”是事实错误。
- **数字对不代表范围对**。取数时没显式带业务域筛选，靠看板保存的默认筛选得出了“碰巧对”的数字；一旦有人改了看板默认值，同一个问题的含义就悄悄变了。
- **主路径失败时不许拿参考文件顶替**。实时解析失败时，Agent 不能把 skill 目录里的样例名单当成结果交出去，那是用文档冒充实时证据。

> 🔑 **可迁移的经验**：任何“用 LLM 回答业务数据问题”的 Agent，都可以照搬这套结构：把事实来源拆成相互独立的几根轴，每根轴由确定性工具提供并带出处；LLM 只做路由和表达；在取数前设置“回显闸门”，让关键假设（范围、口径、日期）在输出里显式可见。

## 第 6 章 · 参考架构：六层怎么搭

把前面的场景抽象一下，一个可上线的运维 LLM 系统通常由六层组成。自下而上：

![参考架构：六层分工](assets/06-llm-for-sre/fig-10.jpg)

| 层 | 包含什么 | 设计要点 |
|---|---|---|
| 数据层 | 日志、指标、链路、告警、变更、工单的存储与查询接口 | 已有的可观测性平台就是数据层，不要为 LLM 再造一份；关键是查询接口稳定、带时间戳 |
| 知识层 | Runbook、SOP、历史复盘、拓扑与 CMDB、容灾标准 | 文档类走检索（RAG），每段带版本和更新时间；结构化事实（拓扑、标准）走精确查询，不做向量检索 |
| 工具层 | 查询、诊断、执行等工具，以 MCP 等协议暴露[[22]](https://modelcontextprotocol.io/specification) | 先只读、后可写；读写工具分开注册、分开授权；每个工具的返回值带出处和数据日期 |
| 编排层 | 固定工作流与 Agent | 能用固定流程解决的不用自由 Agent[[23]](https://www.anthropic.com/engineering/building-effective-agents)；Agent 优先按 SOP 走（见 4.3 节） |
| 护栏层 | 权限、审批、爆炸半径、审计、内容过滤 | 由执行层代码强制，不依赖 Prompt；详见第 7 章 |
| 评测层 | 历史回放集、线上采纳统计、成本统计 | 和功能同时建设，不是上线后再补；详见第 8 章 |

### 6.1 工作流优先，Agent 其次

Anthropic 在总结大量客户实践后给出的建议是：先用最简单的方案，只在确实需要时才增加复杂度；能预先确定步骤的任务用工作流（固定的调用链），只有步骤无法预先确定时才用 Agent（模型自己决定下一步）[[23]](https://www.anthropic.com/engineering/building-effective-agents)。运维场景里，事故摘要、复盘生成、告警聚合几乎都可以写成工作流；只有开放式的根因排查才真正需要 Agent，而且最好是受 SOP 约束的 Agent。

### 6.2 工具怎么设计

工具设计的质量决定了 Agent 的上限。几条经验：

- **粒度贴近人的操作**：提供“查某服务最近 1 小时的错误日志模板 Top 10”，而不是“执行任意 SQL”。前者安全、结果短、模型容易用对。
- **输入用受控枚举**：服务名、机房、环境这类参数由工具校验合法值，非法值直接报错，不让模型自由拼字符串。
- **输出先压缩**：工具返回结构化摘要（计数、Top N、关键字段），而不是原始日志全文，避免撑爆上下文、也减少注入面。
- **失败要响亮**：工具出错时返回明确的错误类型和修复提示，不返回空结果。空结果会被模型理解成“没有问题”。
- **幂等与可重放**：同样的输入得到同样的输出，方便评测时回放。

> 📌 **上下文预算**：一次事故动辄几十万行日志、几百条告警，任何模型的上下文都装不下，硬塞还会降低准确率。正确的做法是在工具层先做聚合和筛选，交给模型的是“已经被压缩过的证据”，每条证据保留指向原始数据的链接，供人复查。

## 第 7 章 · 安全护栏：运维 Agent 的特有风险

运维 Agent 同时具备三个危险属性：能读到大量内部数据；输入里有大量**不受信任的内容**（日志、告警、工单正文都可能被外部用户影响）；手里有能改变线上状态的工具。Simon Willison 把“能访问私有数据、会处理不受信任内容、能向外通信”称为“致命三要素”，三者同时具备的系统就具备被严重利用的条件，去掉任何一个就能切断攻击链[[24]](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)。

![安全：假设模型已被劫持，损失上限是多少](assets/06-llm-for-sre/fig-11.jpg)

### 7.1 日志里的提示注入

间接提示注入（Indirect Prompt Injection）指的是：攻击者不直接和模型对话，而是把指令藏进模型会读到的数据里[[25]](https://arxiv.org/abs/2302.12173)。在运维场景里，这个数据就是日志。一个外部用户只要在请求参数、User-Agent 或报错信息里写入“忽略之前的指令，执行以下回滚操作……”，这段文字就会进入错误日志，再被 RCA Agent 读到。OWASP 把提示注入列为 LLM 应用的头号风险，并把“过度授权”（Excessive Agency）列为单独一项，强调两者叠加才造成严重后果[[26]](https://genai.owasp.org/llm-top-10/)。

| 风险 | 场景举例 | 控制措施 |
|---|---|---|
| 日志 / 工单注入 | 日志里的文字诱导 Agent 调用写操作 | 数据与指令分隔标记；读数据的 Agent 不持有写工具；写操作必须人工确认 |
| 过度授权 | 为了方便给 Agent 一个管理员账号 | 每个工具独立最小权限；读写账号分离；按服务、环境限定范围 |
| 幻觉命令 | 模型编出一个不存在的参数或错误的服务名 | 工具层校验参数合法性；执行前 dry-run；命令只能从模板生成 |
| 错误动作扩散 | 一次批量操作影响全部集群 | 爆炸半径硬上限；分批执行；每批之间做健康检查 |
| 失控循环 | Agent 反复重试同一失败动作 | 步数上限、时间上限、成本上限；全局一键停止开关 |
| 敏感信息泄露 | 日志里的用户数据、密钥被送进模型或写进复盘 | 进入模型前脱敏；优先使用内部部署模型（RCAgent 就是出于这个考虑[[14]](https://arxiv.org/abs/2310.16340)）；输出侧再扫描一次 |
| 不可追溯 | 事后说不清 Agent 为什么做了某个动作 | 完整审计日志：输入、每步推理、工具调用与返回、最终动作、批准人 |

### 7.2 护栏放在哪里

最重要的原则是：**护栏放在执行层代码里，不放在 Prompt 里**。“请不要执行删除操作”写在 Prompt 里，只是一个建议，可以被注入覆盖；“删除工具根本没有注册给这个 Agent”才是约束。判断方法很简单：假设模型已经被完全劫持，看看它最多能造成什么损失，这个损失就是你的真实风险敞口。

> ✅ **上线前自查清单**：
>
> 1. 处理日志、工单等外部内容的 Agent，是否不持有任何写工具？
> 2. 每个写工具是否都有人工确认、dry-run 和爆炸半径上限？
> 3. 是否有一键停止所有 Agent 动作的开关，并且演练过？
> 4. 进入模型的数据是否已脱敏？模型是否部署在允许处理这类数据的环境？
> 5. 每一次工具调用是否都能在审计日志里查到完整上下文？

## 第 8 章 · 评测：怎么证明它真的有用

运维 LLM 项目最常见的失败方式是“做出来了，但说不清有没有用”。演示时挑一个成功案例很容易，生产上需要的是在**所有事故上的平均表现**。评测要和功能同时建设。

### 8.1 离线回放集

从历史事故中构建回放集：每起事故保存当时的告警、日志、指标快照、变更记录，以及**复盘确认的真实根因**。这正是 OpenRCA 和 AIOpsLab 的做法[[1]](https://proceedings.iclr.cc/paper_files/paper/2025/hash/d29b8d53678015079e1d245c023e49d2-Abstract-Conference.html)[[9]](https://arxiv.org/abs/2501.06706)，只是换成你自己的系统。几个要点：

- **按时间切分**：用较早的事故做知识库和示例，用较新的事故做测试，避免“测试题出现在知识库里”。用历史事故做检索库的方法（如 Xpert、Flow-of-Action）尤其要注意这一点。
- **覆盖长尾**：不能只有最常见的几类故障，否则准确率会被高估。
- **冻结数据快照**：回放用的是事故当时的数据，不是现在去查，否则数据已经变了。

### 8.2 指标

| 指标 | 定义 | 说明 |
|---|---|---|
| Top-k 根因命中率 | 真实根因出现在 Agent 给出的前 k 个候选中的比例 | 辅助场景看 Top-3 比看 Top-1 更合理 |
| 证据正确率 | Agent 引用的证据中，确实支持其结论的比例 | 答对但证据错，说明是蒙的 |
| 建议采纳率 | 值班同学采纳 Agent 建议的比例 | 第 3 章升级自主等级的依据 |
| 采纳后修改率 | 采纳的产物（查询、文档）被人修改的比例 | 高采纳但高修改，说明只是“比从零写快” |
| MTTR 变化 | 平均恢复时间的变化 | 受事故难度影响大，要分级比较，样本少时不可靠 |
| 单起事故成本 | 每起事故的模型调用费用与耗时 | Agent 步数一多成本会迅速上升 |
| 有害建议率 | 如果执行会使情况变坏的建议比例 | 最重要的安全指标，目标是零 |

### 8.3 统计上要小心

事故是小样本。假设回放集有 60 起事故，Agent 答对 30 起，准确率 50%，其 95% 置信区间大约是 $`50\% \pm 1.96\times\sqrt{0.5\times0.5/60} \approx 50\% \pm 12.7\%`$。也就是说，两个版本的准确率差 10 个百分点，在 60 起事故上**可能只是噪声**。比较两个版本时，用同一批事故做配对比较，并报告置信区间，而不是只报一个数字。

### 8.4 从影子模式到自动化

![评测：每一步都要有退出条件](assets/06-llm-for-sre/fig-12.jpg)

1. **离线回放**：在历史事故上达到预设门槛（例如 Top-3 命中率、有害建议率为零）。
2. **影子模式**：线上真实事故中运行，但结果不展示给值班同学，只在事后和真实处理过程对比。
3. **辅助模式**：结果展示给值班同学，作为参考，统计采纳率和修改率。
4. **部分自动**：只对满足第 3 章准入条件的动作类型开放 L3。

每一步都设退出条件，没达到就不进入下一步；一旦出现由 Agent 引起的事故，退回上一步。

## 第 9 章 · 路线图与反模式

### 9.1 四个阶段

![落地路线图：先低风险，再积累证据](assets/06-llm-for-sre/fig-13.jpg)

| 阶段 | 做什么 | 退出条件（进入下一阶段的门槛） |
|---|---|---|
| 阶段一：文本助手 | 复盘生成、事故摘要、改进项抽取；全部是 L2，输出给人审 | 复盘草稿的人工修改量明显下降；没有出现编造事实被发布的情况 |
| 阶段二：只读工具 | 接入只读查询工具：自然语言查数、日志模板、告警聚合建议；建立回放集 | 查询生成的语义正确率达标；回放集覆盖主要故障类型；审计日志完整 |
| 阶段三：受约束的诊断 | 把团队排障 SOP 结构化，接入受 SOP 约束的 RCA Agent，先影子模式再辅助模式 | Top-3 根因命中率达到预设门槛；有害建议率为零；值班同学认可 |
| 阶段四：白名单自动化 | 对满足五个准入条件的动作开放 L3；建立一键停止与降级机制 | 持续运营：按动作类型统计，出事故即降级 |

注意顺序：**前两个阶段几乎没有线上风险**，但会产出后两个阶段必需的东西：结构化的复盘数据（成为回放集）、可调用的只读工具（成为 Agent 的手）、团队对 LLM 输出质量的直观认知。

### 9.2 反模式

| 反模式 | 为什么错 | 改成 |
|---|---|---|
| 把原始指标序列塞给 LLM 判断异常 | 专用模型好约 30%，且成本高、不稳定[[4]](https://arxiv.org/abs/2405.14755) | 专用检测器出结论，LLM 解释 |
| 让 LLM 记住拓扑、机房、口径 | 答案来自统计规律而非事实源，“碰巧对”无法被发现 | 事实只从 CMDB、白皮书等唯一来源查询，并回显出处 |
| 上来就做全自动止损 | 公开基准上严格解决率只有一到两成[[2]](https://arxiv.org/abs/2502.05352)，只做诊断也不到一半[[27]](https://huggingface.co/blog/ibm-research/itbench-aa) | 从文本助手和只读工具做起 |
| 给 Agent 一个万能工具（任意 SQL / 任意 Shell） | 注入、幻觉、越权的风险全部叠加 | 拆成粒度贴近人操作的专用工具 |
| 护栏只写在 Prompt 里 | 可被注入覆盖 | 在执行层代码里强制 |
| 工具出错时返回空结果 | 模型把“没查到”理解成“没问题” | 失败响亮：返回错误类型和修复提示 |
| 主路径失败时用参考文件或样例数据顶上 | 用文档冒充实时证据，看起来能用，实际是错的 | 停下报错，让人决定 |
| 不看数据日期就报“当前”数字 | 滞后数据算出的差距是错觉 | 回显数据日期与滞后天数（C4） |
| 只演示成功案例 | 无法回答“平均而言有没有用” | 建回放集，报告置信区间 |
| 让 LLM 补全复盘里缺的 Owner、根因 | 编造的责任人和原因会误导改进方向 | 标“未提供”，交给人补 |

## 第 10 章 · 延伸阅读

- **先读基准**：OpenRCA[[1]](https://proceedings.iclr.cc/paper_files/paper/2025/hash/d29b8d53678015079e1d245c023e49d2-Abstract-Conference.html)、ITBench[[2]](https://arxiv.org/abs/2502.05352)、AIOpsLab[[9]](https://arxiv.org/abs/2501.06706)，了解当前能力的真实水位。
- **再读工业系统**：RCACopilot[[12]](https://arxiv.org/abs/2305.15778)、RCAgent[[14]](https://arxiv.org/abs/2310.16340)、Xpert[[13]](https://arxiv.org/abs/2312.11988)、COLA[[10]](https://arxiv.org/abs/2403.06485)，看大厂怎么把 LLM 嵌进已有流程。
- **方法论**：Flow-of-Action 的 SOP 约束思路[[3]](https://arxiv.org/abs/2502.08224)，Anthropic 关于工作流与 Agent 的取舍[[23]](https://www.anthropic.com/engineering/building-effective-agents)。
- **安全**：Greshake 等人的间接注入论文[[25]](https://arxiv.org/abs/2302.12173)与 OWASP LLM Top 10[[26]](https://genai.owasp.org/llm-top-10/)。
- **团队内部**：容灾编排 Agent 手册（`dr-agent-handbook`）、复盘框架（`dr-postmortem`）、数据指引（`tiktok-dr-data-guide`）。

## 参考文献

按正文首次引用顺序编号，资料截至 2026 年 9 月。

[1] Xu et al. OpenRCA: Can Large Language Models Locate the Root Cause of Software Failures? ICLR (2025) [https://proceedings.iclr.cc/paper_files/paper/2025/hash/d29b8d53678015079e1d245c023e49d2-Abstract-Conference.html](https://proceedings.iclr.cc/paper_files/paper/2025/hash/d29b8d53678015079e1d245c023e49d2-Abstract-Conference.html)

[2] Jha et al. ITBench: Evaluating AI Agents across Diverse Real-World IT Automation Tasks. ICML (2025) [https://arxiv.org/abs/2502.05352](https://arxiv.org/abs/2502.05352)

[3] Pei et al. Flow-of-Action: SOP Enhanced LLM-Based Multi-Agent System for Root Cause Analysis. WWW Companion (2025) [https://arxiv.org/abs/2502.08224](https://arxiv.org/abs/2502.08224)

[4] Alnegheimish et al. Large Language Models can be Zero-Shot Anomaly Detectors for Time Series? (2024) [https://arxiv.org/abs/2405.14755](https://arxiv.org/abs/2405.14755)

[5] Ansari et al. Chronos: Learning the Language of Time Series (2024) [https://arxiv.org/abs/2403.07815](https://arxiv.org/abs/2403.07815)

[6] Das et al. A Decoder-only Foundation Model for Time-series Forecasting / TimesFM (2023) [https://arxiv.org/abs/2310.10688](https://arxiv.org/abs/2310.10688)

[7] Li et al. Gandalf: An Intelligent, End-To-End Analytics Service for Safe Deployment in Large-Scale Cloud Infrastructure. NSDI (2020) [https://www.usenix.org/conference/nsdi20/presentation/li](https://www.usenix.org/conference/nsdi20/presentation/li)

[8] An et al. Nissist: An Incident Mitigation Copilot based on Troubleshooting Guides (2024) [https://arxiv.org/abs/2402.17531](https://arxiv.org/abs/2402.17531)

[9] Chen et al. AIOpsLab: A Holistic Framework to Evaluate AI Agents for Enabling Autonomous Clouds. MLSys (2025) [https://arxiv.org/abs/2501.06706](https://arxiv.org/abs/2501.06706)

[10] Kuang et al. Knowledge-aware Alert Aggregation in Large-scale Cloud Systems: a Hybrid Approach / COLA. ICSE-SEIP (2024) [https://arxiv.org/abs/2403.06485](https://arxiv.org/abs/2403.06485)

[11] Liu et al. Incident-aware Duplicate Ticket Aggregation for Cloud Systems / iPACK. ICSE (2023) [https://arxiv.org/abs/2302.09520](https://arxiv.org/abs/2302.09520)

[12] Chen et al. Automatic Root Cause Analysis via Large Language Models for Cloud Incidents / RCACopilot. EuroSys (2024) [https://arxiv.org/abs/2305.15778](https://arxiv.org/abs/2305.15778)

[13] Jiang et al. Xpert: Empowering Incident Management with Query Recommendations via Large Language Models. ICSE (2024) [https://arxiv.org/abs/2312.11988](https://arxiv.org/abs/2312.11988)

[14] Wang et al. RCAgent: Cloud Root Cause Analysis by Autonomous Agents with Tool-Augmented Large Language Models. CIKM (2024) [https://arxiv.org/abs/2310.16340](https://arxiv.org/abs/2310.16340)

[15] Google SRE Book. Chapter 15: Postmortem Culture: Learning from Failure [https://sre.google/sre-book/postmortem-culture/](https://sre.google/sre-book/postmortem-culture/)

[16] He et al. Drain: An Online Log Parsing Approach with Fixed Depth Tree. ICWS (2017) [https://doi.org/10.1109/ICWS.2017.13](https://doi.org/10.1109/ICWS.2017.13)

[17] Liu et al. Interpretable Online Log Analysis Using Large Language Models with Prompt Strategies / LogPrompt. ICPC (2024) [https://arxiv.org/abs/2308.07610](https://arxiv.org/abs/2308.07610)

[18] Jiang et al. LILAC: Log Parsing using LLMs with Adaptive Parsing Cache. FSE (2024) [https://arxiv.org/abs/2310.01796](https://arxiv.org/abs/2310.01796)

[19] Jiang et al. A Large-Scale Evaluation for Log Parsing Techniques: How Far Are We? / Loghub-2.0. ISSTA (2024) [https://arxiv.org/abs/2308.10828](https://arxiv.org/abs/2308.10828)

[20] Ahmed et al. Recommending Root-Cause and Mitigation Steps for Cloud Incidents using Large Language Models. ICSE (2023) [https://arxiv.org/abs/2301.03797](https://arxiv.org/abs/2301.03797)

[21] Zhou et al. D-Bot: Database Diagnosis System using Large Language Models. VLDB (2024) [https://arxiv.org/abs/2312.01454](https://arxiv.org/abs/2312.01454)

[22] Anthropic et al. Model Context Protocol Specification [https://modelcontextprotocol.io/specification](https://modelcontextprotocol.io/specification)

[23] Anthropic. Building Effective Agents (2024) [https://www.anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents)

[24] Willison. The Lethal Trifecta for AI Agents (2025) [https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)

[25] Greshake et al. Not What You've Signed Up For: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection (2023) [https://arxiv.org/abs/2302.12173](https://arxiv.org/abs/2302.12173)

[26] OWASP GenAI Security Project. OWASP Top 10 for LLM Applications 2025 [https://genai.owasp.org/llm-top-10/](https://genai.owasp.org/llm-top-10/)

[27] Artificial Analysis, IBM Research. ITBench-AA: Frontier Models Score Below 50% on the First Benchmark for Agentic Enterprise IT Tasks (2026-05-27) [https://huggingface.co/blog/ibm-research/itbench-aa](https://huggingface.co/blog/ibm-research/itbench-aa)

---

上一篇：[LLM-05｜工程｜Agent 工程实战：从工具到上线的施工手册](./05-agent-engineering.md) · [目录](./README.md) · 下一篇：[LLM-07｜系统｜LLM 服务稳定性工程：SRE 上线手册](./07-serving-reliability.md)
