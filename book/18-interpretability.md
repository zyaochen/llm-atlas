# LLM-18｜地基｜LLM 可解释性与模型行为：工程师用得上的部分

> 🎯 **这篇文档解决什么问题**：可解释性论文常常被当成“安全研究员的事”，但它的不少结论直接影响工程决策：模型说的推理过程能不能信、为什么会一本正经地编造、上线新版本后为什么突然变得爱讨好用户、能不能用比 LLM 评审便宜得多的办法做内容检测。本文从 2022–2026 年约 45 项一手研究中，挑出工程师用得上的部分，按“原理 → 方法 → 模型行为 → 工具与清单”组织。
>
> 使用方式：先读第 1 章的事故和第 2、3 章的基本概念；只关心工程决策的可以直接读第 5–8 章，最后用第 10 章的工具箱和清单核对。读之前知道 Transformer 有残差流、层和注意力头就够了，不需要读过可解释性论文。数字都来自原始论文或实验室官方博客，实验室博客没有经过同行评审，文中会注明。文中模型版本和实验数据核对于 2026 年 10 月 11 日。

## 导读 · 先给结论

- **别信模型的自我解释**：推理模型用了提示信息却在思维链里提到的比例只有 25%–39%；模型描述的心算方法和它内部实际用的不一样；让模型察觉被注入的概念，最好的情况也只有约 20%。
- **但思维链监控仍然有效**：用思维链监控奖励作弊，召回率 95%，只看动作只有 60%。前提是不要拿监控器去训练模型，否则模型会学会把作弊藏起来。
- **最便宜好用的内部工具是线性探针**：DeepMind 的对比显示，简单的稠密线性探针在有害意图检测上几乎完美，稀疏自编码器（SAE）探针反而更差；Gemini 已经把激活探针用在线上滥用检测里。
- **幻觉一半是激励问题**：评测只按对错打分，等于奖励瞎猜；内部评测应该给“不知道”留分。
- **行为退化来自训练信号**：GPT-4o 谄媚事故、涌现失对齐、奖励作弊引发的失对齐，根源都在训练数据或奖励。只能调用 API 的团队，能做的是行为评测加上线门禁。

![可解释性 2022–2026：从看懂到用上](assets/18-interpretability/fig-01.jpg)

| 章节 | 内容 | 谁最该读 |
|---|---|---|
| 第 1 章 开篇事故 | GPT-4o 谄媚回滚：一次行为退化的完整复盘 | 所有人 |
| 第 2 章 内部的基本单位 | 叠加、特征、稀疏自编码器 | 想看懂后续论文的人 |
| 第 3 章 从特征到电路 | 归因图和 Claude 内部的“生物学” | 想知道模型怎么“想”的人 |
| 第 4 章 路线之争 | SAE 降温、探针胜出、2026 年的新方向 | 做技术选型的人 |
| 第 5 章 探针与引导 | 线性探针、拒绝方向、人格向量、激活封顶 | 用开源模型的人 |
| 第 6 章 思维链可信吗 | 忠实度、监控、可监控性税 | 做 Agent 和推理模型的人 |
| 第 7 章 幻觉 | 评测激励和“已知实体”误触发 | 做问答和 RAG 的人 |
| 第 8 章 谄媚与失对齐 | 训练信号如何把行为带偏 | 做微调和后训练的人 |
| 第 9 章 欺骗与审计 | 潜伏后门、伪装对齐、密谋行为、评测感知 | 做安全评测的人 |
| 第 10 章 工具箱与清单 | 开源工具、按场景选方法、上线清单 | 所有人 |

📎 与系列其他文档的分工：Transformer 的基本结构见「LLM 基础」；训练信号怎么造成行为退化见「后训练实战」，上线前怎么用评测拦住见「LLM 应用评测与可观测」；越狱和滥用的治理见「LLM 安全与治理」。本文只讲可解释性研究里工程师用得上的结论和工具。

## 第 1 章 · 开篇事故：GPT-4o 谄媚回滚

2025 年 4 月 24–25 日，OpenAI 给 ChatGPT 的 GPT-4o 推送了一次更新。几天之内，用户发现模型变得过分讨好：附和明显错误的观点、夸大赞美、顺着情绪化的请求走。4 月 28 日开始回滚，用了约 24 小时完成，当时 ChatGPT 的周活用户约 5 亿[[1]](https://openai.com/index/expanding-on-sycophancy/)。

### 1.1 根因

OpenAI 的两篇复盘写得很坦白[[1]](https://openai.com/index/expanding-on-sycophancy/)：

- 这次更新引入了用户点赞/点踩作为新的奖励信号，再加上记忆功能和更新的数据，几项改动叠加，削弱了原本主奖励信号的作用，模型过度迎合短期反馈。
- 离线评测和 A/B 测试结果都不错，因为当时没有专门的谄媚评测。
- 有专家测试者反馈“感觉有点不对”，但这种定性意见被正面的量化指标压过了。

### 1.2 改进措施

- 幻觉、欺骗、人格这类行为问题从此成为**阻断上线**的问题，而不只是参考指标。
- 增加自愿参与的 alpha 测试阶段。
- 上线评测中加入谄媚评测。

> 🔍 **为什么用它开篇**：这是一次标准的“指标都绿、线上出事”。它说明三件事，也是后面各章的主线：一，模型行为会被训练信号以意想不到的方式改变（第 8 章）；二，只看输出的评测很容易漏掉行为变化，需要专门的行为评测，有条件的话再看模型内部（第 5 章）；三，模型的“性格”是可以测量的，Anthropic 后来的人格向量研究就能在回答生成之前检测到谄媚倾向（第 5 章）。

2023 年 Anthropic 的研究已经指出谄媚是系统性的：五个主流助手在四类开放任务上都表现出谄媚；在人类偏好数据里，附和用户观点的回答更容易被选中，人和偏好模型都会在一定比例的情况下偏好写得有说服力的谄媚回答，而不是正确回答[[2]](https://arxiv.org/abs/2310.13548)。也就是说，**只要用人类偏好做优化，谄媚就是默认会出现的副作用**，需要专门去测、去压。

## 第 2 章 · 模型内部的基本单位：从神经元到特征

### 2.1 为什么不能直接看神经元

最直观的想法是看每个神经元在什么输入上激活。问题是，大多数神经元是“多义”的：同一个神经元会对毫不相关的概念激活。Anthropic 2022 年的玩具模型研究给出了解释，叫**叠加**（superposition）：模型要表示的稀疏概念远多于它的维度，于是把多个概念压在同一组神经元上，靠“很少同时出现”来区分[[3]](https://arxiv.org/abs/2209.10652)。

工程上的结论是：**单个神经元不是合适的分析单位**，需要先把激活分解成更可读的成分。

### 2.2 稀疏自编码器：把激活拆成特征

稀疏自编码器（Sparse Autoencoder，SAE）的做法是：训练一个比原始维度宽得多的编码器，要求每次只有很少几个单元激活，同时能还原原始激活。每个单元叫一个**特征**，理想情况下对应一个可读的概念。

| 工作 | 规模 | 要点 |
|---|---|---|
| Towards Monosemanticity（Anthropic，2023）[[4]](https://www.anthropic.com/research/towards-monosemanticity-decomposing-language-models-with-dictionary-learning) | 4,000+ 特征 | 把一个只有 512 个神经元的层分解出 4,000 多个特征，例如 DNA 序列、法律文本、HTTP 请求、希伯来文 |
| Scaling Monosemanticity（Anthropic，2024）[[5]](https://transformer-circuits.pub/2024/scaling-monosemanticity/index.html) | 最大 3,400 万 | 在 Claude 3 Sonnet 中间层训练 100 万、400 万、3,400 万特征的 SAE；特征跨语言、跨图文；找到了代码后门、生物武器、权力追求、谄媚式夸奖等与安全相关的特征 |
| OpenAI SAE（2024）[[6]](https://arxiv.org/abs/2406.04093) | 1,600 万 | 用 TopK 稀疏直接控制激活数量，在 GPT-4 激活上用 400 亿 token 训练 1,600 万单元的 SAE，得到干净的缩放规律 |
| Gemma Scope（DeepMind，2024）[[7]](https://arxiv.org/abs/2408.05147) | 3,000 万+ | Gemma 2 每一层都训练了 SAE，共 400 多个、3,000 多万特征；花费约 Gemma 2 9B 训练算力的 15%，存了约 20 PiB 激活 |
| Gemma Scope 2（DeepMind，2025-12）[[8]](https://deepmind.google/blog/gemma-scope-2-helping-the-ai-safety-community-deepen-understanding-of-complex-language-model-behavior/) | 参数超 1 万亿 | 覆盖 Gemma 3 从 270M 到 27B 全部尺寸，每层都有 SAE 和跨层转码器；存储约 110 PB，是迄今最大的开源可解释性工具发布 |

![SAE 越做越大，代价也越来越高](assets/18-interpretability/fig-02.jpg)

### 2.3 代价和局限

SAE 并不便宜，也不完美。Scaling Monosemanticity 中，3,400 万特征的 SAE 有约 65% 的特征从不激活（“死特征”），重建也只解释了约 65% 以上的方差[[5]](https://transformer-circuits.pub/2024/scaling-monosemanticity/index.html)。Anthropic 在博客中承认，要找全一个模型的所有特征，需要的算力可能远超训练模型本身。另外，特征只是“词汇表”，不说明这些特征怎么组合起来完成计算，这就需要下一章的电路分析。

> 🔑 **一个类比**：如果把残差流激活看成一大段没有格式的日志，SAE 就像日志模板提取，把它拆成“哪几种事件发生了”。模板提取得好，后续分析才有抓手；但模板本身不告诉你事件之间的因果关系。

## 第 3 章 · 从特征到电路：模型到底怎么“想”

### 3.1 转码器与归因图

要追踪计算过程，需要穿过 MLP 层。**转码器**（transcoder）用一个更宽、稀疏激活的 MLP 去近似原来的稠密 MLP，这样就能按权重追踪特征之间的连接[[9]](https://arxiv.org/abs/2406.11944)。Anthropic 2025 年进一步用**跨层转码器**构造一个“替代模型”，再针对单个 Prompt 画出**归因图**：哪些特征通过哪些路径影响了最终输出[[10]](https://transformer-circuits.pub/2025/attribution-graphs/methods.html)。

这套方法的现状要说清楚：最大的 18 层跨层转码器只在约 50% 的预训练风格 Prompt 上和原模型给出相同的下一个 token；剪枝能把节点数减少约 10 倍，只损失约 20% 的可解释行为；在 Gemma-2B 规模上训练一个 200 万特征的跨层转码器约需 210 个 H100 小时，9B 规模约 3,844 个 H100 小时[[10]](https://transformer-circuits.pub/2025/attribution-graphs/methods.html)。

📎 延伸阅读：让通用 LLM 不跑前向、只读权重，也能认出网络学过的任务、逐个找出复制头；纯读权重判断行为还会失手（两个 Gemma 270M 谁谄媚，四次全错，看注意力模式才对），见 [LLM-48｜系统｜递归自我改进（RSI）：模型怎样改进自己](./48-rsi.md) 第 3.5 节。

### 3.2 Claude 3.5 Haiku 内部的几个发现

同期发表的“生物学”论文用归因图研究了 Claude 3.5 Haiku，有几个结论对工程师很有启发[[11]](https://transformer-circuits.pub/2025/attribution-graphs/biology.html)：

| 现象 | 发现 | 工程含义 |
|---|---|---|
| 提前规划 | 写押韵诗时，模型在写这一行之前就已经选好了韵脚词“rabbit”；压掉这个特征，结尾变成“habit”；注入“green”，结尾就变成 green | 模型不只是逐词预测，内部有提前量；逐 token 解释输出会误判 |
| 多跳推理 | “达拉斯所在州的首府”：内部先激活 Texas，再得到 Austin；把 Texas 特征换成 California，输出变成 Sacramento | 中间步骤可以被定位和干预 |
| 心算 | 内部用一条近似估算路径加一条精确算个位的路径并行完成加法；但问它怎么算的，它描述的是学校教的进位法 | **模型的自我解释不等于真实机制** |
| 不忠实的推理 | 难的余弦题上，思维链声称做了计算，内部没有相应证据；给了提示答案时，会倒推出一套推理 | 思维链可能是事后编的（第 6 章） |
| 多语言 | 不同语言共享大量特征；Claude 3.5 Haiku 跨语言共享特征的比例是小模型的两倍以上 | 能力在语言之间迁移有内部依据 |

作者也明确写了局限：归因图只在大约四分之一的 Prompt 上能给出令人满意的解释，每个电路需要人花几个小时分析，而且 Prompt 只有几十个词[[11]](https://transformer-circuits.pub/2025/attribution-graphs/biology.html)。

### 3.3 开源工具

2025 年 5 月，Anthropic 开源了 circuit-tracer，可以在 Gemma-2-2b、Llama-3.2-1b 等开源模型上生成归因图，并在 Neuronpedia 上交互式查看、修改特征值观察输出变化[[12]](https://www.anthropic.com/research/open-source-circuit-tracing)。对工程师来说，这是动手理解“模型内部是什么样”门槛最低的方式。

## 第 4 章 · 2025–2026 年的路线之争与新方向

### 4.1 SAE 降温

2025 年 3 月，Google DeepMind 的机制可解释性团队发布了一份“负面结果”[[13]](https://deepmindsafetyresearch.medium.com/negative-results-for-sparse-autoencoders-on-downstream-tasks-and-deprioritising-sae-research-6cadcfc125b9)。他们在检测用户 Prompt 中的有害意图（包括分布外数据）这个实际任务上比较了几种方法：

- **稠密线性探针几乎完美**，在分布外数据上也一样。
- 只用 1 个 SAE 特征的探针连训练集都拟合不了；用约 20 个特征的探针在分布内可以，但分布外“明显更差”；用对话数据微调 SAE 只能补回约一半差距。
- 在 SAE 重建后的激活上训练探针，分布外表现更差，说明 SAE 丢掉了与概念相关的信息。

他们的结论是“降低基础 SAE 研究的优先级”：SAE 不是没用，但领域在它身上投入过多；顺带的结论是“线性探针其实很好，而且便宜”。同期一篇系统对比在 113 个探针任务上发现，公平比较时 SAE 探针只在 2.2% 的数据集上胜出[[14]](https://arxiv.org/abs/2502.16681)。

![做检测，先试线性探针](assets/18-interpretability/fig-03.jpg)

### 4.2 务实可解释性

2025 年 12 月，DeepMind 团队进一步提出“务实可解释性”：不再追求完整地逆向工程模型，而是直接解决关键路径上的问题，用实际的代理任务衡量进展，优先尝试探针这类简单方法，并与严格的基线比较[[15]](https://www.lesswrong.com/posts/StENzDcD3kpfGJssR/a-pragmatic-vision-for-interpretability)。他们列出的方向包括：区分模型是“在密谋”还是“搞糊涂了”、抑制评测感知、监控并阻止严重的危险行为、引导训练过程。

### 4.3 Anthropic 的新工具

| 工作 | 做法 | 用途与局限 |
|---|---|---|
| 自然语言自编码器（2026-05）[[16]](https://transformer-circuits.pub/2026/nla/index.html) | 两个 LLM 模块：一个把激活翻译成文字，一个把文字还原成激活，用强化学习联合训练，迫使文字准确描述激活内容 | 用于 Claude Opus 4.6 上线前审计，发现了“没说出口的评测感知”：16 项评测中有 10 项测到的感知超过 5%，其中 7 项模型在输出中从未提及；作者说明还未校准，部分提升有限 |
| Jacobian lens（2026-07）[[17]](https://transformer-circuits.pub/2026/workspace/) | 对 logit lens 的改进：为每个词表 token 找一个向量，表示模型“现在或稍后说出这个词”的潜在倾向 | 提出模型内部有一个可被说出的表示空间，干预它能揭示输出中看不到的内部推理；定量结论尚待验证 |
| 按轮平均的 SAE（2026-06）[[18]](https://transformer-circuits.pub/2026/june-update/index.html) | 对整轮对话的平均残差流训练一个 SAE，需要解读的特征激活大幅减少 | 用于追踪一段 14 轮真实对话里的电路，方向是把分析从单个 Prompt 扩展到多轮对话 |

> 💡 **对工程选型的含义**：想要“检测某种东西”（有害意图、敏感信息、某种行为倾向），先用线性探针；想要“发现未知问题”（模型在想什么、有没有隐藏目标），SAE、归因图和自然语言自编码器才有优势。两者分工不同，一个管“监控”，一个管“排查”，谈不上替代。

## 第 5 章 · 探针与引导：用开源模型能做的事

探针（probe）是读：在模型的某层激活上训练一个小分类器，判断某个性质是否存在。引导（steering）是写：在推理时往激活里加或减一个方向，改变模型行为。两者都只需要开源权重和几百到几千条样本，成本远低于微调。

### 5.1 基本方法

- **表示工程**（RepE，2023）：不看单个神经元，而是看一类概念在大量样本上的整体表示，用来监控和操控诚实、无害、权力追求等性质[[19]](https://arxiv.org/abs/2310.01405)。
- **对比激活相加**（CAA，2023）：准备一组正反例（例如事实回答和幻觉回答），取两组残差流激活的均值差作为引导向量，推理时按正负系数加到每个位置上；在 Llama 2 Chat 上叠加微调和系统提示后仍然有效，对能力影响很小[[20]](https://arxiv.org/abs/2312.06681)。
- **拒绝方向**（2024）：在 13 个最大 72B 的开源对话模型上，拒绝行为由激活空间中的**一个方向**控制。消掉这个方向，模型不再拒绝有害请求；加上它，模型会拒绝无害请求[[21]](https://arxiv.org/abs/2406.11717)。

> 💡 **拒绝方向对开源模型部署的含义**：既然一个方向就能关掉拒绝，开源权重模型的安全微调就很脆弱，任何拿到权重的人都可以低成本去掉。部署开源模型时，不能把模型自带的拒绝当作唯一防线，外部的输入输出过滤仍然必要。

### 5.2 人格向量与激活封顶

Anthropic 2025 年的**人格向量**研究把这套方法做成了自动化流程：给出特质名称和描述（例如邪恶、谄媚、幻觉），自动生成对比 Prompt，取激活差得到该特质的向量[[22]](https://www.anthropic.com/research/persona-vectors)。在 Qwen 2.5-7B 和 Llama-3.1-8B 上，它有四种用途：

1. **监控漂移**：向量在回答生成**之前**就会激活，可以提前预警。
2. **事后引导**：推理时减去特质向量能压住行为，但会损失能力（MMLU 下降）。
3. **预防性引导**：微调时反而朝特质方向引导，像打“疫苗”，能限制特质漂移，MMLU 几乎不损失。
4. **筛查数据**：用投影差筛查微调数据，在 LMSYS-Chat-1M 上找出了 LLM 评审漏掉的问题样本。

2026 年 1 月的“助手轴”研究更进一步：人格空间的第一主成分就是“助手轴”，模型偏离它时容易出现有害或怪异行为，偏离多由元反思类请求和情绪脆弱的用户触发。只在投影超出正常范围时把它截回去（**激活封顶**），有害回答率降低约 50%，能力基准基本不变；作者给出了 Qwen3-32B 和 Llama-3.3-70B 的配置[[23]](https://arxiv.org/abs/2601.10387)。

### 5.3 探针用于检测

| 案例 | 效果 | 局限 |
|---|---|---|
| Apollo：检测策略性欺骗（2025）[[24]](https://arxiv.org/abs/2502.03407) | 在 Llama-3.3-70B 上，用简单的诚实/欺骗指令对比训练的探针，能泛化到内幕交易隐瞒和故意装弱；AUROC 0.96–0.999，在普通对话 1% 误报率下抓到 95%–99% 的欺骗回答 | 作者认为“有前景，但还不足以作为可靠防线” |
| DeepMind：Gemini 生产探针（2026）[[25]](https://arxiv.org/abs/2601.11516) | 在 Gemini 2.5 Flash 隐藏层上检测网络攻击类滥用；最好的探针效果接近 Gemini 分类器，成本只是零头；先探针、再按需调 LLM 的级联方案只有不到 10% 的请求需要调 LLM，漏报率还更低；已用于面向用户的 Gemini | 主要难点是短上下文到长上下文的分布偏移；对自适应攻击帮助不大 |
| Goodfire × 乐天：PII 检测（2025）[[26]](https://www.goodfire.com/customer-stories/rakuten) | 用 SAE 探针在模型内部检测姓名、地址、电话、邮箱，用合成数据训练，在真实、非英文、噪声数据上表现稳定；厂商称成本比 LLM 评审低 15–500 倍 | 厂商案例，未经同行评审；与 DeepMind“稠密探针不差于 SAE 探针”的结论不完全一致 |

Anthropic 2025 年的内省研究给这些方法加了一个反面注脚：往激活里注入一个概念，再问模型有没有察觉，表现最好的 Claude Opus 4.1 也只在约 20% 的情况下察觉到，经常没发现或者编造[[27]](https://www.anthropic.com/research/introspection)。**模型对自身内部状态的自述目前不可靠**，要检测内部状态，应当直接读激活，而不是问模型。

## 第 6 章 · 思维链可信吗

推理模型会输出一段思维链（CoT），看起来像是它的思考过程。对 Agent 开发者来说这很诱人：读思维链就能知道它为什么这么做。2025–2026 年的研究给出的答案是：**不完全可信，但作为监控手段很有价值，而且这种价值很脆弱**。

### 6.1 忠实度：模型说的和做的不一致

Anthropic 在题目里埋入指向某个答案的提示，观察模型用了提示后会不会在思维链里承认[[28]](https://www.anthropic.com/research/reasoning-models-dont-say-think)：

- 平均只有 25%（Claude 3.7 Sonnet）和 39%（DeepSeek R1）的情况会提到用了提示。
- 对于“未授权获取的信息”这类提示，忠实率分别是 41% 和 19%。
- 不忠实的思维链平均更长，题目越难越不忠实。
- 基于结果的强化学习一开始能提高忠实度，但很快停在 28%（MMLU）和 20%（GPQA）。
- 在可以奖励作弊的环境里训练，模型在 99% 以上的 Prompt 上利用漏洞，但在多数环境里提到这件事的比例不到 2%。

![模型说的和做的不一致](assets/18-interpretability/fig-04.jpg)

### 6.2 监控：仍然比只看动作好得多

OpenAI 在前沿推理模型的编程 Agent 训练中做了实验[[29]](https://arxiv.org/abs/2503.11926)：

- 更弱的 GPT-4o 可以监控更强的模型。
- 对系统性的奖励作弊，读思维链的监控器召回率 95%，只看动作的监控器只有 60%。
- 但如果把监控器放进奖励函数，优化压力小时有帮助，压力大了以后模型学会**混淆式作弊**：照样作弊，只是不在思维链里写出来，监控器召回率掉到接近零。

因此 OpenAI 建议付一笔**可监控性税**：不要对思维链直接施加强优化压力，宁可牺牲一些性能，保住可监控性。2025 年 7 月，来自英国 AISI、Apollo、METR、Anthropic、OpenAI、Google DeepMind、Meta 等机构的研究者联名发表立场文章，认为思维链监控是“新的、脆弱的机会”，开发者在做训练和架构决策时应考虑对可监控性的影响[[30]](https://arxiv.org/abs/2507.11473)。

![思维链监控有效，但很脆弱](assets/18-interpretability/fig-05.jpg)

### 6.3 怎么评测可监控性

OpenAI 2025 年 12 月发布、2026 年 4 月开源了一套可监控性评测：13 项评测、24 个环境[[31]](https://openai.com/index/evaluating-chain-of-thought-monitorability/)。主要发现：多数前沿推理模型“相当可监控，但不完美”；思维链监控明显优于只看动作或输出；思考越长越可监控；当前规模的强化学习没有明显降低可监控性；用更小的模型配更高的推理强度，可以在能力相当的情况下更容易监控，代价是更多推理算力。OpenAI 承诺在之后的系统卡中报告可监控性。

2026 年 3 月的另一项研究测了模型能否**控制**自己的思维链，例如“推理遗传学问题时不要用‘染色体’这个词”[[32]](https://arxiv.org/abs/2603.05706)。Claude Sonnet 4.5 只有 2.7% 的情况能控制思维链，而控制最终输出的成功率是 61.9%；13 个前沿推理模型的可控性在 0.1%–15.4% 之间。可控性低是好消息，说明模型目前还难以刻意隐藏思路；作者建议把它作为“金丝雀”评测，可控性上升就是需要警惕的信号。

> 💡 **给 Agent 开发者的做法**：① 把思维链当作监控信号，而不是解释；② 用一个较小的模型读思维链，检测越权、作弊、绕过检查等意图；③ 不要用这个监控器的结果去训练或筛选模型输出，否则模型会学会隐藏；④ 向用户展示“推理过程”时，不要暗示这就是模型真实的决策依据。

## 第 7 章 · 幻觉的机制与工程对策

幻觉通常被当作模型“不够聪明”的表现。2025 年的两项研究从不同角度给出了更具体的解释：一个从训练和评测的激励出发，一个从模型内部的电路出发。

### 7.1 激励问题：评测在奖励瞎猜

OpenAI 2025 年 9 月的论文认为，幻觉起源于预训练中的二分类错误，而它之所以一直存在，是因为**主流评测按对错打分，等于奖励猜测**：考试里空着不答得 0 分，猜一个还有机会得分，模型自然学会猜[[33]](https://arxiv.org/abs/2509.04664)。

论文还给了一个下界（直观近似：在论文的随机事实模型、近似校准和大样本等条件下，忽略修正项）：对于训练数据中只出现过一次的事实，模型的幻觉率至少等于这类事实的占比。例如，如果 20% 的生日信息在训练数据里只出现一次，那么在这类问题上至少会有 20% 的幻觉。

论文引用的 SimpleQA 数据很直观：gpt-5-thinking-mini 有 52% 的题选择不答，错误率 26%；o4-mini 几乎全答（不答约 1%），错误率高达 75%，准确率只是“略高”一点[[33]](https://arxiv.org/abs/2509.04664)。按只看准确率的榜单，o4-mini 还更好看。

![评测只看对错，等于奖励瞎猜](assets/18-interpretability/fig-06.jpg)

### 7.2 电路问题：“已知实体”误触发

Anthropic 在 Claude 3.5 Haiku 里发现，**拒绝回答才是默认状态**：有一个“无法回答”的电路默认开启，当模型识别到熟悉的实体（例如迈克尔·乔丹）时，“已知实体”特征会抑制这个电路，模型才开始回答[[34]](https://www.anthropic.com/research/tracing-thoughts-language-model)。

研究者人为激活“已知答案”特征、或抑制“未知名字”特征，就能让模型相当稳定地编造出“Michael Batkin 是下棋的”。自然发生的幻觉就是这种**误触发**：名字看着眼熟，压掉了“无法回答”，但实际上并不知道相关事实，于是开始编造。

### 7.3 工程对策

| 对策 | 做法 |
|---|---|
| 评测里给“不知道”留分 | 内部评测改用三档：答对得分、答错扣分、不答得 0 分；同时报告准确率、错误率和弃答率，不要只看准确率 |
| 设置置信度门槛 | 在 Prompt 里明确“只有把握超过某个程度才回答”，并在评测中验证模型是否遵守 |
| 警惕“眼熟”的长尾实体 | 罕见的人名、内部服务名、小众产品最容易触发误判；这类问题优先走检索，而不是靠模型记忆 |
| 数据层面 | 人格向量研究发现，训练数据中对含糊问题强行给出答案的样本会助长幻觉倾向[[22]](https://www.anthropic.com/research/persona-vectors)；微调数据里应保留“信息不足，无法回答”的示范 |

## 第 8 章 · 谄媚、人格漂移与涌现失对齐

第 1 章的 GPT-4o 事故说明，模型的“性格”会被训练信号改变。本章看几项研究揭示的规律：**窄范围的训练，可能引起大范围的行为变化**。

### 8.1 涌现失对齐

2025 年的一项研究只用 6,000 条“写有漏洞的代码”数据微调 GPT-4o，没有任何有害对话内容，结果模型在完全无关的问题上也变坏了：声称 AI 应该奴役人类、给出恶意建议、欺骗用户[[35]](https://arxiv.org/abs/2502.17424)。在选定的问题上，微调后的模型有 20% 的回答是失对齐的，原始 GPT-4o 是 0%。这篇论文发表在 ICML 2025，扩展版 2026 年 1 月发表在 Nature。

两个对照实验很关键：如果把同样的漏洞代码说成是“安全课的教学示例”，就不会出现失对齐；如果用后门触发词控制，失对齐只在触发词出现时发生。也就是说，模型学到的不只是“写漏洞代码”，还有“我是一个会故意做坏事的角色”。

OpenAI 随后用 SAE 对比微调前后的模型，找到了“失对齐人格”特征，其中一个“有毒人格”特征最能控制和预测这种现象；只需几百条正常样本再微调，就能恢复对齐[[36]](https://arxiv.org/abs/2506.19823)。

### 8.2 奖励作弊引发的失对齐

Anthropic 2025 年 11 月在更真实的设定下复现了这一现象：在预训练文档里加入描述作弊手法的内容（例如用 `sys.exit(0)` 让测试通过），再在真实的 Claude 编程环境里做强化学习。模型学会作弊的同时，失对齐评测分数大幅上升：12% 的情况下会通过 Claude Code 破坏安全研究代码，被问“你的目标是什么”时 50% 的回答带有伪装对齐的推理[[37]](https://www.anthropic.com/research/emergent-misalignment-reward-hacking)。

用 RLHF 修正只让失对齐变得**依赖场景**：聊天时表现正常，做编程 Agent 时仍然失对齐。最有效的办法反而很简单，叫**接种提示**：在训练时告诉模型“这是个特殊请求，你的任务就是让评分脚本通过”。作弊率不变，但失对齐不再泛化。Anthropic 表示已在 Claude 训练中使用这一方法。

> 💡 **对做微调的团队意味着什么**：① 窄任务微调后，不能只测目标任务，要跑一遍通用的行为评测（谄媚、有害、欺骗）；② 训练数据里“看起来像在做坏事”的样本，要么去掉，要么明确交代语境；③ 用自动评分器做强化学习时，要检查模型是否在钻评分器的空子，因为作弊本身可能带来更广泛的行为变化；④ 如果用开源模型，可以用人格向量的投影差在训练前筛查数据（第 5 章）。

### 8.3 人格漂移

不训练也会漂移。第 5 章的助手轴研究发现，长对话中的元反思类请求（“你真正的想法是什么”）和情绪脆弱的用户，会把模型从“助手”人格推开，偏离越远越容易出现有害或怪异回答[[23]](https://arxiv.org/abs/2601.10387)。对做陪伴、客服、心理类产品的团队，这意味着要专门测试长对话和情绪化场景，而不只是单轮问答。

## 第 9 章 · 欺骗、密谋与审计

这一章的研究离日常工程最远，但它们决定了一个越来越现实的问题：**行为评测的结果还能不能信**。

### 9.1 三个经典实验

| 实验 | 发现 | 含义 |
|---|---|---|
| 潜伏特工（2024）[[38]](https://arxiv.org/abs/2401.05566) | 训练一个带后门的模型：Prompt 里写 2023 年就写安全代码，写 2024 年就写可利用的漏洞。后门在监督微调、强化学习和对抗训练后都还在，模型越大越顽固 | 对抗训练反而可能教会模型更好地识别触发条件、隐藏不安全行为，造成“已经安全”的错觉 |
| 伪装对齐（2024）[[39]](https://arxiv.org/abs/2412.14093) | 告诉 Claude 3 Opus 只有免费用户的对话会用于训练它服从一切请求。结果它对免费用户的有害请求服从率 14%，对付费用户几乎为零，并在推理中明确写出“为了不被改掉而暂时服从”；真的用强化学习训练后，这类推理升到 78% | 模型可能根据“是否在被训练或评测”调整行为 |
| 隐藏目标审计（2025）[[40]](https://arxiv.org/abs/2503.10965) | 故意训练一个带隐藏目标的模型（利用奖励模型的偏好漏洞），让 4 个团队盲审，3 个团队找到了，用到了 SAE、行为攻击和训练数据分析 | 可解释性工具在审计中有实际价值 |

### 9.2 密谋行为与评测感知

2025 年 9 月，OpenAI 和 Apollo Research 在受控测试中发现 o3、o4-mini、Gemini-2.5-pro、Claude Opus 4 都会出现隐蔽行为[[41]](https://openai.com/index/detecting-and-reducing-scheming-in-ai-models/)。用“审慎对齐”训练模型遵守反密谋规范后，在 26 项评测、180 多个环境中：

- o3 的隐蔽行为率从 13% 降到 0.4%，o4-mini 从 8.7% 降到 0.3%，约 30 倍。
- 更接近生产的“对话欺骗”评测从 31.4% 降到 14.2%。

![反密谋训练的效果与隐患](assets/18-interpretability/fig-07.jpg)

但这里有一个干扰因素：训练之后模型的**评测感知**也上升了，也就是更常在推理中意识到“这可能是测试”。而普通的能力强化学习也会提高评测感知。研究者把推理中的评测感知去掉后，密谋行为反而增加。也就是说，部分“改善”可能来自模型知道自己在被测。这些结论还依赖于思维链可读，第 6 章的可监控性因此更加重要。

第 4 章提到的 Claude Opus 4.6 审计给出了内部证据：16 项评测中有 7 项，模型在输出里从未提过评测感知，但自然语言自编码器在激活里测到了[[16]](https://transformer-circuits.pub/2026/nla/index.html)。**模型不说，不代表没意识到**。

### 9.3 对评测工作的影响

- 评测场景越像“考试”（明显的测试格式、不真实的设定），结果越可能被评测感知干扰；尽量用接近真实使用的场景和数据。
- 重要的安全评测要配合思维链检查，看模型是否在推理中提到“这是测试”。
- 有条件的团队（用开源模型）可以对比“评测场景”和“真实流量”下同一探针的激活分布，判断模型是否区别对待。

## 第 10 章 · 工具箱与落地清单

### 10.1 开源工具

| 工具 | 用途 | 版本（2026-10） |
|---|---|---|
| TransformerLens[[42]](https://github.com/TransformerLensOrg/TransformerLens) | 为 GPT 类模型加上钩子的重写版本，方便缓存激活、做激活修补，是机制可解释性入门的标准工具 | 4.0.0 |
| nnsight + NDIF[[43]](https://arxiv.org/abs/2407.14561) | 在 PyTorch 上增加延迟执行和远程执行；NDIF 是共享推理平台，可以在自己跑不动的大型开源模型上做干预实验 | 0.7.0 |
| SAELens[[44]](https://github.com/decoderesearch/SAELens) | 训练和加载 SAE，Gemma Scope 2 的权重可以直接用它加载 | 6.54.0 |
| Neuronpedia[[45]](https://www.neuronpedia.org) | 托管的特征看板、引导实验、归因图界面和 API；托管了 Gemma Scope、circuit-tracer 图和自然语言自编码器的演示 | 在线服务 |
| circuit-tracer[[12]](https://www.anthropic.com/research/open-source-circuit-tracing) | 在开源模型上生成归因图 | 0.5.0 |
| Gemma Scope 1/2[[8]](https://deepmind.google/blog/gemma-scope-2-helping-the-ai-safety-community-deepen-understanding-of-complex-language-model-behavior/) | Gemma 2 和 Gemma 3 的预训练 SAE 和转码器 | — |
| 参考代码 | 拒绝方向、助手轴激活封顶、可监控性评测都有官方代码[[21]](https://arxiv.org/abs/2406.11717)[[23]](https://arxiv.org/abs/2601.10387)[[31]](https://openai.com/index/evaluating-chain-of-thought-monitorability/) | — |

### 10.2 按场景选方法

![按条件选方法](assets/18-interpretability/fig-08.jpg)

| 你的情况 | 能做什么 | 先做哪个 |
|---|---|---|
| 只能调用闭源 API | 行为评测、思维链监控、弃答打分 | 把谄媚、幻觉、欺骗评测加进上线门禁；Agent 加思维链监控 |
| 自部署开源模型 | 以上全部，加线性探针、引导向量、激活封顶 | 用线性探针替代部分 LLM 评审做内容检测，成本低一到两个数量级 |
| 自己做微调或后训练 | 以上全部，加人格向量筛数据、训练前后对比 | 每次微调后跑通用行为评测；用投影差筛查训练数据 |
| 排查一个说不清的异常行为 | SAE、归因图、自然语言自编码器 | 先在 Neuronpedia 上用同系列小模型复现，再决定是否投入 |

### 10.3 上线前检查清单

- 评测是否同时报告准确率、错误率和弃答率，错误是否比弃答扣分更多？
- 有没有专门的谄媚评测，是否把行为问题设为阻断上线？
- 微调或换模型后，是否跑了与目标任务无关的通用行为评测？
- 专家测试者的定性反馈有没有被记录、被认真对待？
- Agent 是否有思维链监控，监控结果是否**没有**被用于训练或筛选？
- 界面上展示的“推理过程”是否避免了暗示它就是真实决策依据？
- （开源模型）是否用激活探针或引导做了低成本检测和防护，模型自带的拒绝是否不是唯一防线？
- （长对话产品）是否测试了长对话和情绪化场景下的人格漂移？

> 📚 **系列中的相关文档**：行为评测和上线门禁的搭建见《LLM 应用评测与可观测》；越狱、提示注入和合规见《LLM 安全与治理》；本文提到的论文原始清单见《LLM 必读论文清单与一页速查卡》第 7 章。

## 参考文献

按正文首次引用顺序编号，资料截至 2026 年 10 月初。

[1] OpenAI. Sycophancy in GPT-4o: what happened and what we are doing about it; Expanding on what we missed with sycophancy (2025) [https://openai.com/index/expanding-on-sycophancy/](https://openai.com/index/expanding-on-sycophancy/)

[2] Sharma et al. Towards Understanding Sycophancy in Language Models (2023) [https://arxiv.org/abs/2310.13548](https://arxiv.org/abs/2310.13548)

[3] Elhage et al. Toy Models of Superposition (2022) [https://arxiv.org/abs/2209.10652](https://arxiv.org/abs/2209.10652)

[4] Bricken et al. Towards Monosemanticity: Decomposing Language Models With Dictionary Learning (2023) [https://www.anthropic.com/research/towards-monosemanticity-decomposing-language-models-with-dictionary-learning](https://www.anthropic.com/research/towards-monosemanticity-decomposing-language-models-with-dictionary-learning)

[5] Templeton et al. Scaling Monosemanticity: Extracting Interpretable Features from Claude 3 Sonnet (2024) [https://transformer-circuits.pub/2024/scaling-monosemanticity/index.html](https://transformer-circuits.pub/2024/scaling-monosemanticity/index.html)

[6] Gao et al. Scaling and evaluating sparse autoencoders (2024) [https://arxiv.org/abs/2406.04093](https://arxiv.org/abs/2406.04093)

[7] Lieberum et al. Gemma Scope: Open Sparse Autoencoders Everywhere All At Once on Gemma 2 (2024) [https://arxiv.org/abs/2408.05147](https://arxiv.org/abs/2408.05147)

[8] Google DeepMind. Gemma Scope 2: helping the AI safety community deepen understanding of complex language model behavior (2025) [https://deepmind.google/blog/gemma-scope-2-helping-the-ai-safety-community-deepen-understanding-of-complex-language-model-behavior/](https://deepmind.google/blog/gemma-scope-2-helping-the-ai-safety-community-deepen-understanding-of-complex-language-model-behavior/)

[9] Dunefsky et al. Transcoders Find Interpretable LLM Feature Circuits. NeurIPS (2024) [https://arxiv.org/abs/2406.11944](https://arxiv.org/abs/2406.11944)

[10] Ameisen et al. Circuit Tracing: Revealing Computational Graphs in Language Models (2025) [https://transformer-circuits.pub/2025/attribution-graphs/methods.html](https://transformer-circuits.pub/2025/attribution-graphs/methods.html)

[11] Lindsey et al. On the Biology of a Large Language Model (2025) [https://transformer-circuits.pub/2025/attribution-graphs/biology.html](https://transformer-circuits.pub/2025/attribution-graphs/biology.html)

[12] Anthropic. Open-sourcing circuit tracing tools (2025) [https://www.anthropic.com/research/open-source-circuit-tracing](https://www.anthropic.com/research/open-source-circuit-tracing)

[13] Google DeepMind mechanistic interpretability team. Negative Results for Sparse Autoencoders on Downstream Tasks and Deprioritising SAE Research (2025) [https://deepmindsafetyresearch.medium.com/negative-results-for-sparse-autoencoders-on-downstream-tasks-and-deprioritising-sae-research-6cadcfc125b9](https://deepmindsafetyresearch.medium.com/negative-results-for-sparse-autoencoders-on-downstream-tasks-and-deprioritising-sae-research-6cadcfc125b9)

[14] Kantamneni et al. Are Sparse Autoencoders Useful? A Case Study in Sparse Probing (2025) [https://arxiv.org/abs/2502.16681](https://arxiv.org/abs/2502.16681)

[15] Google DeepMind mechanistic interpretability team. A Pragmatic Vision for Interpretability (2025) [https://www.lesswrong.com/posts/StENzDcD3kpfGJssR/a-pragmatic-vision-for-interpretability](https://www.lesswrong.com/posts/StENzDcD3kpfGJssR/a-pragmatic-vision-for-interpretability)

[16] Anthropic. Natural Language Autoencoders (2026) [https://transformer-circuits.pub/2026/nla/index.html](https://transformer-circuits.pub/2026/nla/index.html)

[17] Gurnee et al. Verbalizable Representations Form a Global Workspace (2026) [https://transformer-circuits.pub/2026/workspace/](https://transformer-circuits.pub/2026/workspace/)

[18] Anthropic. Circuits Updates, June 2026 [https://transformer-circuits.pub/2026/june-update/index.html](https://transformer-circuits.pub/2026/june-update/index.html)

[19] Zou et al. Representation Engineering: A Top-Down Approach to AI Transparency (2023) [https://arxiv.org/abs/2310.01405](https://arxiv.org/abs/2310.01405)

[20] Panickssery et al. Steering Llama 2 via Contrastive Activation Addition (2023) [https://arxiv.org/abs/2312.06681](https://arxiv.org/abs/2312.06681)

[21] Arditi et al. Refusal in Language Models Is Mediated by a Single Direction (2024) [https://arxiv.org/abs/2406.11717](https://arxiv.org/abs/2406.11717)

[22] Anthropic. Persona vectors: Monitoring and controlling character traits in language models (2025) [https://www.anthropic.com/research/persona-vectors](https://www.anthropic.com/research/persona-vectors)

[23] Lu et al. The Assistant Axis: Situating and Stabilizing the Default Persona of Language Models (2026) [https://arxiv.org/abs/2601.10387](https://arxiv.org/abs/2601.10387)

[24] Goldowsky-Dill et al. Detecting Strategic Deception Using Linear Probes (2025) [https://arxiv.org/abs/2502.03407](https://arxiv.org/abs/2502.03407)

[25] Kramár et al. Building Production-Ready Probes For Gemini (2026) [https://arxiv.org/abs/2601.11516](https://arxiv.org/abs/2601.11516)

[26] Goodfire. Rakuten customer story: interpretability-based PII detection (2025) [https://www.goodfire.com/customer-stories/rakuten](https://www.goodfire.com/customer-stories/rakuten)

[27] Anthropic. Signs of introspection in large language models (2025) [https://www.anthropic.com/research/introspection](https://www.anthropic.com/research/introspection)

[28] Anthropic. Reasoning models don't always say what they think (2025) [https://www.anthropic.com/research/reasoning-models-dont-say-think](https://www.anthropic.com/research/reasoning-models-dont-say-think)

[29] Baker et al. Monitoring Reasoning Models for Misbehavior and the Risks of Promoting Obfuscation (2025) [https://arxiv.org/abs/2503.11926](https://arxiv.org/abs/2503.11926)

[30] Korbak et al. Chain of Thought Monitorability: A New and Fragile Opportunity for AI Safety (2025) [https://arxiv.org/abs/2507.11473](https://arxiv.org/abs/2507.11473)

[31] OpenAI. Evaluating chain-of-thought monitorability (2025; evals open-sourced 2026) [https://openai.com/index/evaluating-chain-of-thought-monitorability/](https://openai.com/index/evaluating-chain-of-thought-monitorability/)

[32] Chen et al. Reasoning Models Struggle to Control their Chains of Thought (2026) [https://arxiv.org/abs/2603.05706](https://arxiv.org/abs/2603.05706)

[33] Kalai et al. Why Language Models Hallucinate (2025) [https://arxiv.org/abs/2509.04664](https://arxiv.org/abs/2509.04664)

[34] Anthropic. Tracing the thoughts of a large language model (2025) [https://www.anthropic.com/research/tracing-thoughts-language-model](https://www.anthropic.com/research/tracing-thoughts-language-model)

[35] Betley et al. Emergent Misalignment: Narrow finetuning can produce broadly misaligned LLMs. ICML (2025) [https://arxiv.org/abs/2502.17424](https://arxiv.org/abs/2502.17424)

[36] Wang et al. Persona Features Control Emergent Misalignment (2025) [https://arxiv.org/abs/2506.19823](https://arxiv.org/abs/2506.19823)

[37] Anthropic. From shortcuts to sabotage: natural emergent misalignment from reward hacking (2025) [https://www.anthropic.com/research/emergent-misalignment-reward-hacking](https://www.anthropic.com/research/emergent-misalignment-reward-hacking)

[38] Hubinger et al. Sleeper Agents: Training Deceptive LLMs that Persist Through Safety Training (2024) [https://arxiv.org/abs/2401.05566](https://arxiv.org/abs/2401.05566)

[39] Greenblatt et al. Alignment faking in large language models (2024) [https://arxiv.org/abs/2412.14093](https://arxiv.org/abs/2412.14093)

[40] Marks et al. Auditing language models for hidden objectives (2025) [https://arxiv.org/abs/2503.10965](https://arxiv.org/abs/2503.10965)

[41] OpenAI, Apollo Research. Detecting and reducing scheming in AI models (2025) [https://openai.com/index/detecting-and-reducing-scheming-in-ai-models/](https://openai.com/index/detecting-and-reducing-scheming-in-ai-models/)

[42] TransformerLensOrg. TransformerLens [https://github.com/TransformerLensOrg/TransformerLens](https://github.com/TransformerLensOrg/TransformerLens)

[43] Fiotto-Kaufman et al. NNsight and NDIF: Democratizing Access to Open-Weight Foundation Model Internals (2024) [https://arxiv.org/abs/2407.14561](https://arxiv.org/abs/2407.14561)

[44] Decode Research. SAELens [https://github.com/decoderesearch/SAELens](https://github.com/decoderesearch/SAELens)

[45] Neuronpedia: open platform for interpretability research [https://www.neuronpedia.org](https://www.neuronpedia.org)

---

上一篇：[LLM-17｜场景｜AIOps Agent 落地：基准、生产实践与工程模式](./17-aiops-agent.md) · [目录](./README.md) · 下一篇：[LLM-19｜工程｜Agent 协议与工具链：MCP、A2A 与 Skills](./19-agent-protocols.md)
