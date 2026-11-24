# LLM-48｜系统｜递归自我改进（RSI）：模型怎样改进自己

> 🎯 **这篇文档解决什么问题**：RSI（recursive self-improvement）被讲了很多年，但 2023 年之后真正落地的工作分布在一个二维谱上：改的对象（脚手架代码、权重、policy 加奖励、整段 codebase、裁判、idea 流水线）与判据来源（环境 reward、verifier、LLM 裁判、人类）各不相同，能不能算“递归自改”也分几档。工程师的实在问题是：STOP、Self-Rewarding、SEAL、AlphaEvolve、Agent Lightning 这些到底改了什么、拿数字说话、能不能复现、红线在哪；自己想做一条自改进回路时该选哪一档。
>
> 使用方式：做选型的读者先读导读和第 7、9、11 章，想做没有黄金判据的闭环回路再读第 12 章；想动手复现的读第 3 章（含 3.5 节 LLM 直接改权重）和第 5、10 章；只关心红线的读第 9 章。文中基准数字尽量标注是否为厂商自报；2026-10-16 事实审计发现原 [8] 号引文错配（见第 10.2 节），相关“独立复现”表述已撤回，引用前仍建议回到原文核对。版本、价格和基准分数核对于 2026 年 10 月 15 日。

## 导读 · 先给结论

- **RSI 包含多种机制，分布在“改的对象 × 判据来源”二维里的不同象限**。STOP 改脚手架代码但冻结权重，自己声明“不是完整的递归自改进”[[1]](https://arxiv.org/abs/2310.02304)；SEAL 改的是权重，靠 ReSTEM^EM 而非 GRPO[[2]](https://arxiv.org/abs/2506.10943)；Self-Rewarding 同时改 policy 和隐式奖励函数[[3]](https://arxiv.org/abs/2401.10020)。
- **真正“权重或代码被自身循环更新”的只有少数几条线**。SEAL 用权重级 ReSTEM^EM[[2]](https://arxiv.org/abs/2506.10943)、Self-Rewarding 用 policy 加 Iterative DPO[[3]](https://arxiv.org/abs/2401.10020)、AlphaEvolve 演化整段 codebase 且判据是客观自动评估器[[4]](https://arxiv.org/abs/2506.13131)、Darwin-Gödel Machine 与 HGM 改 Agent 自身源码[[5]](https://arxiv.org/abs/2505.22954)[[6]](https://arxiv.org/abs/2510.21614)、Agent Lightning 把任意 Agent 轨迹 MDP 化后做 policy RL[[7]](https://arxiv.org/abs/2508.03680)。
- **几乎所有基准数字都是厂商自报预印本，本轮未找到可核验的第三方复现**。AlphaEvolve 的 11 维 kissing number 下界（593 个外球）由 DeepMind 公开了构造与校验代码，构造本身可自行验证[[8]](https://github.com/google-deepmind/alphaevolve_results)，但同样属于 DeepMind 自报，本轮没有找到可核验的独立第三方复现；75% 重新发现 SOTA、48 次标量乘法等也只有 DeepMind 自报[[9]](https://deepmind.google/discover/blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/)。
- **自裁判判据有时点风险**。Self-Taught Evaluators 在 RewardBench 上把 Llama-3-70B-Instruct 从 75.4 拉到 88.3（多数投票 88.7），那是 2024-08 的 v1 口径[[10]](https://arxiv.org/abs/2408.02666)；到 2024-10，v1 榜首 Skywork-Reward-Gemma-2-27B-v0.2 已是 Overall 94.3、Chat 子项 96.1[[11]](https://huggingface.co/Skywork/Skywork-Reward-Gemma-2-27B-v0.2)，2025-06 发布的 RewardBench 2 上模型平均分又比 v1 低约 20 分[[42]](https://arxiv.org/abs/2506.01937)，故“匹配顶级奖励模型”只是 v1 时点说法。
- **prompt 层自改容易过拟合，且证据偏负**。仅靠内部知识自纠正会让模型更差[[12]](https://arxiv.org/abs/2310.01798)；在自生成答案上递归再训会出现可测退化，大模型更明显[[13]](https://arxiv.org/abs/2609.34257)。
- **reward overoptimization 与误设是 RSI 的结构性风险，但“自改进循环内出现 spec gaming”已有作者自报案例（DGM 附录 H），尚无独立复现**。过优化曲线[[14]](https://arxiv.org/abs/2210.10760)与奖励误设映射[[15]](https://arxiv.org/abs/2201.03544)是通用背景证据；OpenAI、Anthropic、Google DeepMind 的安全框架都设有 AI 自我改进 / 自主研发能力阈值[[38]](https://cdn.openai.com/pdf/18a02b5d-6b67-4cec-ab64-68cdfbddebcd/preparedness-framework-v2.pdf)[[39]](https://www.anthropic.com/responsible-scaling-policy)[[40]](https://deepmind.google/blog/updating-the-frontier-safety-framework/)，但没有针对自改进回路的专门操作条款（见 9.4）。
- **上线分两档**。vibe RSI（prompt 或示例层自改，便宜、易过拟合）与工程化 RSI（权重或代码级、有客观验证器、可回滚、每轮留评测快照）的取舍见第 11 章。
- **没有黄金判据时，回路跑得完，能力很少留下**。Aspire、S³Gym、HarnessDev 三篇把验收放到 Agent 看不到的地方：30 格实验只有 1 格把增益留到回滚之后，自评认得出好动作却和涨分不相关，外壳在反馈集上涨 13.9 分，到 held-out 只剩 1.43（见第 12 章）。
- **权重级自改多了一条「LLM 直接写权重」的支线，最硬的证据只到移除障碍**。2026 年 11 月的 Weight-Space Programming 报告 GPT-6 Astra 不跑梯度直接读写权重：能从零写出超过 PPO 的小游戏策略，给 Qwen3.5-0.8B 改 0.016% 权重把 MATH-500 从 7.9 拉到 48.1；但增益来自修好思考死循环，换到不循环的 Gemma 上没有可复现提升（见 3.5，作者博客自报，日志已公开[[49]](https://rikiyat.github.io/blog/weight-space-programming/)）。

📎 与系列其他文档的分工：测试时采样、验证器与奖励模型（ORM/PRM/生成式验证器）见「推理模型与测试时计算」第 2–3 章，本文只在判据阶梯里引用一句；GRPO/DAPO 配方与 R1 训练成本见「后训练实战」第 7–9 章，少样本 SFT 见其第 3 章；多 Agent 编排与协同见「多 Agent 系统」整篇，本文只讲“系统改自身某一部分”；模型崩溃触发条件与合成评测自欺见「合成数据」；DSPy/GEPA/ACE 的提示词自动搜索见「提示词与流程的自动优化」，手工工具描述与上下文工程见「Agent 工程实战」第 3–4 章。

## 第 1 章 · RSI 是什么：改的对象与判据来源

递归自我改进（recursive self-improvement，RSI）的严格判据有两条：被改进的对象由系统自身的循环更新（不是人在外面改），并且判据不能退化成“模型自己夸自己”。在 LLM 语境里，这个判据把一大半被叫做“self-improving”的工作挡在门外：它们要么改的是人写的代码，要么权重根本没动。把 2023 年以来的工作摆在一张二维表上，才能看清谁真在自改、改的是什么。

两条轴是：**改的对象**（脚手架代码、模型权重、policy 加奖励函数、整段 codebase、裁判模型、idea 或论文流水线、memory 或评测集），以及**判据来源**（环境 reward、verifier 或 pass@k、LLM 裁判、人类）。Schmidhuber 2003 年的 Gödel Machine 是理论原点：能改写自身任何部分，仅当能形式证明某次自修改严格提升期望未来效用时才应用[[16]](https://arxiv.org/abs/cs/0309048)。今天没有任何系统达到这个可证最优的严格性，HGM 只是其经验近似[[6]](https://arxiv.org/abs/2510.21614)。

📎 展开阅读：「推理模型与测试时计算」第 3 章（ORM/PRM/生成式验证器作为判据的机制细节）、「后训练实战」第 7 章（GRPO/PPO 等策略梯度是否属于“自改”的边界）。

### 1.1 改的对象决定“真 RSI”还是“伪 RSI”

| 系统 | 改的对象 | 判据来源 | 是否真 RSI |
|---|---|---|---|
| STOP | 脚手架/improver 代码 | 环境 utility | 否，权重冻结，自声明[[1]](https://arxiv.org/abs/2310.02304) |
| Self-Rewarding LM | policy 加隐式奖励 | 自身 LLM 裁判 | 是，Iterative DPO 双轴[[3]](https://arxiv.org/abs/2401.10020) |
| SEAL | 权重 | 环境/verifier（下游任务性能） | 是，ReSTEM^EM[[2]](https://arxiv.org/abs/2506.10943) |
| AlphaEvolve | code/算法 | 客观自动评估器 | 是，kissing number 为 DeepMind 自报（构造可校验）[[4]](https://arxiv.org/abs/2506.13131) |
| Self-Taught Evaluators | 裁判 policy | 自产合成对比输出 | 是，无人工标签[[10]](https://arxiv.org/abs/2408.02666) |
| Darwin-Gödel Machine | coding-agent 代码 | 编码基准经验验证 | 是，自代码加归档树[[5]](https://arxiv.org/abs/2505.22954) |
| HGM | coding-agent 代码/policy | 后代聚合基准（CMP） | 是，自代码树搜索[[6]](https://arxiv.org/abs/2510.21614) |
| Agent Lightning | LLM policy 权重 | 任务或环境 reward | 是，轨迹 MDP 化 RL[[7]](https://arxiv.org/abs/2508.03680) |
| AI Scientist v1/v2 | idea/paper 流水线 | 模拟裁判或 workshop 评审 | 否，不改权重或源码[[17]](https://arxiv.org/abs/2408.06292)[[18]](https://arxiv.org/abs/2504.08066) |

### 1.2 判据来源的阶梯

判据越靠客观端，越不容易被“模型自评”污染，但越难覆盖开放任务。环境 reward（AlphaEvolve 的自动评估器、Agent Lightning 的任务成功）最硬，却要求任务可量化；verifier 或 pass@k 见「推理模型与测试时计算」第 3 章，对有标准答案的推理和代码任务有效；LLM 裁判（Self-Rewarding、Self-Taught Evaluators）覆盖面最广但带 self-preference 风险；人类判据最准但不可规模化。第 7 章给每档的实操示例。

![RSI 二维谱：改的对象 × 判据来源，谁算真递归自改](assets/48-rsi/fig-01.jpg)

> 💡 **常见误区**：把“模型给自己打分再训自己”都叫 RSI。如果权重没动（STOP）、或只做 idea 流水线（AI Scientist v1）、或判据是另一个冻结模型，都不到 Gödel Machine 意义上的“自改”。本表里标“否”的系统仍是有用的自改进回路，但它们的失败模式和上限与真 RSI 不同。

## 第 2 章 · 经典谱系：从 Gödel Machine 到 Agent Lightning

RSI 的谱系分两条枝：一条是**推理时自改**，模型在推断时改自己的输出或思路但不改权重；另一条是**权重或代码级自改**，系统的循环真的更新了参数或源码。后者才是 Gödel Machine 严格意义上的延续，前者更多是“self-correction”，在第 8 章会看到它的负证据。

### 2.1 理论原点与 LLM 版尝试

Schmidhuber 2003 年的 Gödel Machine 给出理论标杆：完全自指，能改写自身任何部分（含证明系统与效用函数），仅当能形式证明某自修改严格提升期望未来效用时才应用[[16]](https://arxiv.org/abs/cs/0309048)。今天没有系统达到这个可证最优。Gödel Agent（2024-10）是较直接的 LLM 尝试：一个自指 Agent 读并改写自身运行中的源码，在若干推理与决策基准上改进，但同样没有 Gödel Machine 的形式证明[[19]](https://arxiv.org/abs/2410.04444)。

### 2.2 推理时自改这条枝

STaR（2022-03）让模型自举推理：在能对答案的任务上，用自己生成的合理思路做监督微调，权重会变，但筛选用的是真值答案，判据不在模型内部[[20]](https://arxiv.org/abs/2203.14465)。Quiet-STaR（2024-03）把它推到普通文本：模型在 token 之间生成“安静思考”并据此学习，无需任务专设微调[[21]](https://arxiv.org/abs/2403.09629)。

Self-Refine（2023-03）和 Reflexion（2023-03）则是纯推断时：前者让模型给自己反馈再改输出，后者用语言化的“口头强化”记下失败教训，权重都不动[[22]](https://arxiv.org/abs/2303.17651)[[23]](https://arxiv.org/abs/2303.11366)。SPIN（2024-01）回到权重级，用自博弈（新模型要胜过旧模型）做微调，把弱模型拉强[[24]](https://arxiv.org/abs/2401.01335)。这条枝的共同点是：判据要么是真值、要么是推断时的自我反馈，不是“训练循环内部产生奖励”。

### 2.3 权重与代码级自改这条枝

STOP（2023-10）把改进对象放在脚手架代码：种子 improver 按效用函数多次查询冻结 LM 取最优解，再用 improver 改进 improver 自身；在 GPT-4 上自改进后的程序显著优于种子。但它明确声明“the language models themselves are not altered”，所以不是完整 RSI[[1]](https://arxiv.org/abs/2310.02304)。Self-Rewarding（2024-01，ICML 2024）往前迈一步，让模型自己当 LLM 裁判、在 Iterative DPO 里给自己出奖励，policy 与奖励能力双轴同提升[[3]](https://arxiv.org/abs/2401.10020)。

SEAL（2025-06）把改进对象直接放到权重：self-edit 生成微调数据、持久更新权重，奖励是更新后模型在下游任务的性能；GRPO/PPO 不稳，改用 ReSTEM^EM[[2]](https://arxiv.org/abs/2506.10943)。

同年 UBC / Vector Institute 与 Sakana AI 的 Darwin-Gödel Machine（2025-05）让 编程 Agent 迭代修改自身代码、在编码基准上经验验证，维护一个 Agent 归档长成多样变体树，用经验验证替代可证自改[[5]](https://arxiv.org/abs/2505.22954)；它与 ADAS（2024-08，自动设计 Agent 系统）同出 Jeff Clune 组[[25]](https://arxiv.org/abs/2408.08435)。HGM（2025-10）在同范式上加了一个新判据 CMP：按后代聚合基准性能选变体，而不是按即时分，因为即时分不等于“还能继续改进的能力”[[6]](https://arxiv.org/abs/2510.21614)。

Agent Lightning（2025-08）换了条路：不要求 Agent 自己改源码，而是把任意 Agent 的执行建模成 MDP，用层级式 RL 加 credit-assignment 把轨迹拆成训练转移，更新底层执行 LLM 的权重，判据是任务成功[[7]](https://arxiv.org/abs/2508.03680)。它把“Agent 执行”和“训练”解耦，几乎零代码改动就能接已有 Agent。

![从 Gödel Machine 到 Agent Lightning：两条枝，谁是真 RSI](assets/48-rsi/fig-02.jpg)

> 🔑 谱系里反复出现的三条规律：判据越客观（自动评估器、真值、任务 reward）越不容易被自评污染；权重或代码被自身循环更新才算真 RSI，prompt 级自改只能算 self-correction；经验验证替代可证最优是当前所有实用系统的共同妥协，HGM 的 CMP 是少数试图逼近“还能继续改进”而非“当下最强”的判据。

## 第 3 章 · 改进对象一：policy、planner 与权重

把改进对象放在 policy 或 planner 上，是 RSI 最热闹的一档。它又分两个子档：Self-Rewarding 让模型同时改 policy 与奖励函数；SEAL 直接改权重，奖励来自环境；DGM 与 HGM 改的是 编程 Agent 的 policy 代码本身。

### 3.1 Self-Rewarding：policy 与奖励同轮共进

Self-Rewarding Language Models（Meta/NYU，ICML 2024）让模型自己当 LLM 裁判，在 Iterative DPO 训练中给自己出奖励[[3]](https://arxiv.org/abs/2401.10020)。DPO（直接偏好优化）本身绕过显式奖励模型，在偏好数据上直接改策略[[26]](https://arxiv.org/abs/2305.18290)，Self-Rewarding 把它做成迭代式并让奖励也由同一个模型产生。

论文的关键贡献是证明两条轴能一起涨，绝对分数倒在其次：指令遵循能力，以及“给自己出高质量奖励”的能力。对比之下，标准 RLHF 的冻结奖励模型在训练后无法再改进。判据是模型自身的 LLM 裁判打分，这是“policy × LLM-judge”象限的标准形态，也是 self-preference 风险的来源（第 8 章）。

📎 展开阅读：「后训练实战」第 7 章（GRPO/PPO/DAPO 的策略梯度配方，与 ReSTEM 这类过滤行为克隆的边界）。

📎 自奖励和环境判据都要在训练循环里被反复调用，打分服务和代码沙箱的吞吐会直接卡住整次迭代，这部分怎么设计见第 55 篇 [LLM-55｜系统｜RL 训练系统：rollout 与训练分离、异步与 Agent 环境](./55-rl-systems.md) 第 10 章。

### 3.2 SEAL：权重级，环境判据，ReSTEM^EM

SEAL（MIT，2025-06）把改进对象直接放在权重上：模型生成 self-edit（一段对自身的微调数据），持久更新权重，奖励是更新后模型在下游任务的性能，属于环境或 verifier 判据，不是 LLM 裁判[[2]](https://arxiv.org/abs/2506.10943)。作者试过 GRPO 与 PPO 学 self-edit 生成策略，发现训练不稳，改用 ReSTEM^EM：EM 式过滤行为克隆，拒绝采样加在受奖励的 self-edit 上做 SFT，不是标准策略梯度。ReSTEM 的祖型是 Singh 等的 ReST^EM[[27]](https://arxiv.org/abs/2312.06585)。

在 ARC few-shot（Llama-3.2-1B-Instruct）上：SEAL 72.5% 对比无 RL 的 self-edit 20%、ICL 0%、oracle TTT 上界 100%。在 SQuAD no-passage（Qwen2.5-7B，单段 LoRA）上：SEAL 自生成数据 47.0% 对比 GPT-4.1 合成 46.3%、基模型合成 39.7%、基线 32.7%[[2]](https://arxiv.org/abs/2506.10943)。

两点值得记住。1B 小模型能在 ARC 上从 0% 拉到 72.5%，说明小模型也能跑权重级自改进回路。另外，自生成数据 47.0% 胜过 GPT-4.1 合成的 46.3%，但只领先 0.7 分且无误差棒，不能过度解读。

![SEAL：1B 模型在 ARC 从 0% 到 72.5%，自生成数据胜过 GPT-4.1 合成](assets/48-rsi/fig-03.jpg)

### 3.3 DGM 与 HGM：改 Agent 自己的 policy 代码

Darwin-Gödel Machine（2025-05，UBC / Vector Institute 与 Sakana AI：Zhang、Hu、Lu、Lange、Clune）的改进对象是 编程 Agent 的 Python 代码本身：迭代修改、在编码基准上经验验证每次改动，并维护一个 Agent 归档做开放式分支进化[[5]](https://arxiv.org/abs/2505.22954)。

HGM（2025-10）在同范式上加了一个判据创新 CMP（clade metaproductivity）：按后代聚合基准性能选变体，因为“只按当下基准分选会误导，即时表现不等于还能继续改进的能力”[[6]](https://arxiv.org/abs/2510.21614)。这把 planner 层的自改进从“选最强”推向“选最有潜力”，是目前少数显式建模“改进潜力”而非“即时性能”的工作。

### 3.4 rStar-Math：小模型的自我演化配方

rStar-Math（Microsoft，2025-01）用 MCTS 让小模型自演化深度思考，在数学推理上追平 o1 量级[[28]](https://arxiv.org/abs/2501.04519)。它的判据靠 MCTS 的胜负信号与正确性检查，不是模型自评，落在“policy × verifier”象限。这类工作把 RSI 与「推理模型与测试时计算」的搜索与验证器结合，是 policy 级自改进里数字最实的一支。

> 💡 **常见误区**：把“模型给自己打分”当成 policy 级 RSI 的全部。Self-Rewarding 的自裁判判据带 self-preference；SEAL 用环境判据绕开它但要付“GRPO 不稳、改用 ReSTEM”的代价。选 policy 级回路时先问判据从哪来，再问改没改权重。

### 3.5 让 LLM 直接改权重：Weight-Space Programming

2026 年 11 月，MIT CSAIL 的 Takehi 等人发了一篇博客，报告 GPT-6 Astra 不用梯度、不写训练循环，只读和写权重就能做事[[49]](https://rikiyat.github.io/blog/weight-space-programming/)。作者把这件事叫 Weight-Space Programming：通用 LLM 围着一句自然语言目标直接改权重张量，小网络把权重当文本读，大网络在 Python 内核里用 numpy 操作。先给这条线定档：目前只有一篇博客、一个闭源模型，下面的数字都是作者自报；好在八个实验的完整日志和改过的 Qwen3.5-0.8B 都已公开[[50]](https://github.com/RikiyaT/lm_understand-logs)[[51]](https://huggingface.co/Rikiya/Qwen3.5-0.8B-weight-space-programming)。

| 类别 | 对象与规模 | 条件 | 结果 |
|---|---|---|---|
| 读 | 4 个同结构小游戏策略（1.7 万权重，PPO 训 10 亿步） | 权重以文本给出，不给候选游戏 | 4 个全部认出 |
| 读 | 3 个 716 万参数的字符级 Transformer | 只给中间 4 个 block，输入输出层都拿掉，不许跑网络 | 语言两次全对，复制头与实测逐个一致、零误报 |
| 读 | 两个 Gemma 3 270M，一个谄媚一个诚实 | 只给权重（嵌入层已冻结） | 4 次全失败；放宽到能看注意力模式后才 3 对全中 |
| 修 | Kinetix 小车策略（17 万权重，卡在局部最优） | 一次回复、不用工具，改 3.5% 的权重 | 1024 局从 0 成功变全部成功 |
| 修 | Qwen3.5-0.8B 的思考死循环 | Python 内核改权重，50 次前向，不用梯度，只动 0.016% | MATH-500 7.9 升到 48.1，IFEval 21.0 升到 62.3 |
| 写 | 3 个 MinAtar 游戏策略 | 无数据、无反馈，权重从 0 开始一次写出，打分前没玩过 | 都超过 1 亿步 PPO；Breakout 517.4 对 5 亿步 PPO 的 25.3 |
| 写 | 9×9 围棋网络（148 万权重） | Python 沙箱逐层手写，前向预算 0 到 50 次 | 0 次已能赢小 PPO 过半对局，50 次赢过训 2000 万局的同规模 PPO |

模型的做法和 SEAL 那类权重自改是两条路：它先推理每层该算什么，再翻译成权重。Breakout 网络的第一层算球的落点，第二层把挡板移过去；围棋网络逐层标记棋子和气、沿棋串找危组，零次前向那版就是一次写出的 336 行 Python。修 Qwen 时，它把死循环定位到第 15、19 层 MLP 写出的一个方向，从两层各 64 个输出列里减掉它（共 131,072 个权重），手法和「拒答由单一方向介导」的方向消融同源[[52]](https://arxiv.org/abs/2406.11717)。

同一采样设定（temperature 0.6、top-p 0.95、top-k 20）下，50 次前向后 MATH-500 从 7.9 升到 48.1，GSM8K 从 13.8 升到 67.1，IFEval 从 21.0 升到 62.3，回答撞 32,768 token 上限的比例从 93% 降到 33%[[49]](https://rikiyat.github.io/blog/weight-space-programming/)。折扣有四个。其一，48.1 的增益来自修好思考死循环，并非新增数学能力：基础模型 96 个练习答案里 85 个打满 token 上限；换成很少循环的 Gemma 3 270M，五档前向最多只把 MATH-500 挪动约 1 分（6.8 到 7.7），作者写明改权重能移除挡住能力的障碍，加不出本来没有的能力。其二，基线偏弱：MinAtar 的 PPO 跑到 5 亿步，Breakout 只有 25.3 分，517.4 分的约 20 倍差距有一半要算在基线没训好上。其三，「无环境反馈」只对小实验成立，Qwen 在 10、25 次前向时循环基本没消，50 次才见效。其四，样本极小、只有一个受试模型、博客没有同行评审，4 个游戏、3 个形状、3 种语言都只是个位数样本。

放回本文的二维谱，它占的是「权重 × 外部环境判据」象限里一个新角：执行者从专用优化器换成通用 LLM，改动方向靠推理给出、靠有限次前向验收，和 SEAL 的 ReSTEM 形成对照。作者列出的递归自我改进前景，是模型直接检查、调试自己或后继模型的权重，比写代码起训练任务更原生；同篇还列了免评测审计、零数据反学习、跨架构电路嫁接，目前都还停在设想[[49]](https://rikiyat.github.io/blog/weight-space-programming/)。它和第 8.5 节的理论上限恰好对得上：这篇最硬的结果只做到移除障碍。权重级后门也给第 9 章的审计问题添了一个入口，能读权重就能藏权重，「不跑推理直接从权重查后门」反过来同样成立。

> 💡 **常见误区**：把 MATH-500 从 7.9 到 48.1 读成「LLM 会训模型了」。这次增益主要在修好思考死循环，撞上限率 93% 降到 33%；换到不循环的 Gemma，同一流程没有可复现增益。现阶段读它，重点放在读权重和单方向消融这两个较扎实的能力，别外推到无中生有。

## 第 4 章 · 改进对象二：prompt、context 与 tool 描述

把改进对象放在 prompt、上下文或工具描述上，是最便宜也最容易过拟合的一档。它不动权重，改动可回滚，但判据一旦不硬，回路会很快滑向“在评测集上刷分”。

📎 展开阅读：「提示词与流程的自动优化」整篇（DSPy/MIPRO/SIMBA、GEPA 的反思式变异、ACE 的演化上下文、工具描述自动优化），「Agent 工程实战」第 3–4 章（工具层与上下文工程的手工基本功）。本章只补与 RSI 直接相关的几条线。

### 4.1 Agent Lightning：让现成 Agent 的轨迹可训练

Agent Lightning（Microsoft，2025-08）改的其实是底层执行 LLM 的**权重**，但它对 prompt/tool 层的意义在于“Training-Agent Disaggregation”：把 Agent 执行完全建模成 MDP，用层级式 RL 加 credit-assignment 把任意 Agent 产生的轨迹拆成训练转移，从而与具体 Agent 实现解耦，几乎零代码改动就能接已有 Agent，还能处理多 Agent 与动态工作流[[7]](https://arxiv.org/abs/2508.03680)。

判据是任务成功（环境 reward），不是 LLM 裁判。它让“工具调用与工作流层面的轨迹”变成可被 RL 消化的训练数据，相当于在不改 Agent prompt 结构的前提下改进底层的 policy。

![Agent Lightning：把任意 agent 轨迹 MDP 化，零改动训练底层 policy](assets/48-rsi/fig-04.jpg)

### 4.2 ADAS：让 Agent 自己设计自己的结构

ADAS（2024-08）把改进对象放到 Agent 系统本身：一个元 Agent 发现、组合并改进由代码定义的 Agent 架构，在多任务上经验验证，长出比人工设计更强的 Agent 程序[[25]](https://arxiv.org/abs/2408.08435)。它与 Darwin-Gödel Machine 同出 UBC / Jeff Clune 组，是“Agent 自设计”这条线的早期形态：它改进的是 Agent 的骨架代码，单段 prompt 不在范围内。

这条线后来被 DSPy/GEPA/ACE 在 prompt 与流程层接走（见「提示词与流程的自动优化」第 5、8 章），ADAS 留下的真正 RSI 遗产是“把 Agent 当作可被搜索改写的代码”这个范式。

### 4.3 Eureka：LLM 写并改进奖励函数代码

Eureka（2023-10）让 LLM 用代码写出 RL 的奖励函数，再按奖励梯度做迭代改进，在自由度奖励设计上达到人类水平[[29]](https://arxiv.org/abs/2310.12931)。它的改进对象是**奖励函数代码**（一种特殊的 tool/context），判据是训练出的策略在下游任务的表现。这把 RSI 与“环境 reward”接上：奖励本身被自改进循环更新，而不是冻结。但它仍依赖一个外层任务指标做最终判据，否则奖励代码会滑向“让 Agent 得高分”而非“让 Agent 真变强”。

### 4.4 prompt 与示例层：便宜，但要先问余量

📎 延伸阅读：只改外壳、权重冻结的自我改进，在没有评分的部署环境里怎样找证据，见 [LLM-58｜工程｜Agent 调优：从 trace 找到该改的那一层](./58-agent-tuning.md) 第 6.2 节。

DSPy/GEPA/ACE 在固定评测上自动搜索提示词与流程，本质是“prompt 级自改进”，详见「提示词与流程的自动优化」。在 RSI 语境里只需记住两点：一，它的判据几乎都是一个小评测集，过拟合风险最高，已有工作报告 49% 的运行低于零样本基线（见兄弟篇第 7 章）；二，工具描述的手工迭代与上下文工程是它的地基，「Agent 工程实战」第 3–4 章已展开，本文不重复。

> 💡 **常见误区**：以为“换了 prompt 跑得更好”就是自改进回路在生效。没有留出评测集、没有对照、没有误差棒时，它只是在评测题上过拟合。prompt 级 RSI 必须先做 10 到 20 条留出题的余量测试再决定要不要跑优化器。

## 第 5 章 · 改进对象三：code、algorithm 与 idea 流水线

把改进对象放在 code 或算法上，判据天然最硬：程序能跑、能对答案、能量化。这是 Gödel Machine 原本设想的“改自身代码”最接近的落点，也是结果最容易被外部校验的一条线（但本轮仍未找到可核验的独立第三方复现）。

### 5.1 AlphaEvolve：演化整段 codebase

AlphaEvolve（DeepMind，2025）的自改进循环演化整个 codebase 与算法，不限于单函数：prompt sampler 喂 Gemini，模型生成新程序，自动评估器检查，进化式程序库选变体[[4]](https://arxiv.org/abs/2506.13131)。判据是客观可量化的自动评估器，不是 LLM 裁判。

三个可证伪结果：一，4×4 复数矩阵乘法用 48 次标量乘法，改进 Strassen 1969（递归 2×2 Strassen 对 4×4 为 49）；二，在 50 多个开放数学问题上约 75% 重新发现 SOTA、约 20% 改进已知最优；三，593 个外球建立 11 维 kissing number 新下界（此前纪录 585，自 2013 年起）[[9]](https://deepmind.google/discover/blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/)。

这三个结果的复现状态必须分清：48 次乘法与 75%、20% 的统计是 DeepMind 自报、无第三方复现；kissing number 下界同样是 DeepMind 自报，区别只在于 593 球构造与校验代码已公开，任何人都能验证构造本身[[8]](https://github.com/google-deepmind/alphaevolve_results)；本轮没有找到可核验的独立第三方复现（此前引用的 arXiv:2609.07047 经核验为无关论文，已撤回）。AlphaEvolve 的前身是 FunSearch（Nature 2024），用 LLM 程序搜索在 cap set 等组合问题上找到新发现[[30]](https://www.nature.com/articles/s41586-023-06924-6)。

![AlphaEvolve：三项结果，只有 kissing number 有独立复现](assets/48-rsi/fig-05.jpg)

### 5.2 STOP：code 级但权重冻结

STOP（2023-10）的改进对象也是代码（脚手架/improver 程序），在 GPT-4 上自改进后的程序显著优于种子[[1]](https://arxiv.org/abs/2310.02304)。但它明确声明语言模型本身没被改，因此不算完整 RSI。它和 AlphaEvolve 的差别正是判据硬度：STOP 的效用函数是任务特定的，AlphaEvolve 用的是数学可验证的自动评估器，后者才支持全 codebase 级开放演化。

### 5.3 AI Scientist v1/v2：idea 流水线，不是 code RSI

AI Scientist v1（2024-08）是端到端流水线：生成想法、写代码、跑实验、画图、写完整论文、跑模拟自动评审，后发表于 Nature[[17]](https://arxiv.org/abs/2408.06292)[[31]](https://sakana.ai/ai-scientist-nature/)。它的自改进循环是迭代开放式 ideation，输出与反馈回灌进不断增长的 archive，但原文明确说不更新自身权重、prompt 或核心源码，对自身代码的自指研究列为 future work。v2（2025-04）加了一个 experiment manager Agent 做渐进式 agentic tree search，去掉了对手写模板的依赖[[18]](https://arxiv.org/abs/2504.08066)。

v2 向 ICLR 2025 的 ICBINB（I Can't Believe It's Not Better）workshop 投了 3 篇全自主生成稿件，经组织方同意参与常规双盲评审，评审人只被告知 43 篇投稿中有 3 篇可能由 AI 生成。其中一篇拿到 6 / 7 / 6（均分 6.33，约前 45%），超过该 workshop 接收线，另两篇未达线；按事先约定，该稿在 meta-review 前即撤回，未正式发表[[32]](https://sakana.ai/ai-scientist-first-publication/)[[18]](https://arxiv.org/abs/2504.08066)。

Sakana 自己也注明：3 篇都未达到其内部评估的 ICLR 主会接收标准，workshop 接收率本就明显高于主会。它不改权重或源码，所以落在 idea 流水线象限，不是 code 级 RSI。

> 🔑 code 级 RSI 的判据最硬，也最值得复现：AlphaEvolve 的 kissing number 构造与校验代码已公开，是本领域最容易被外部核对的结果，但本轮未找到可核验的独立第三方复现；其余数字（48 次乘法、75%、20%）同样是 DeepMind 自报。看到这类结果先问“自动评估器可不可重现”，再问“有没有人重现”。

## 第 6 章 · 改进对象四：memory、RAG 与评测集

把改进对象放在 memory、RAG 索引或评测集上，是目前最冷的一档。有人试过，难在判据极难立住：什么算“值得记住”没有客观信号，评测集一旦由同一个模型生成又打分就会自欺。这一章解释为什么它冷，以及在冷区里少数站得住的工作长什么样。

### 6.1 memory 与 RAG：缺稳定判据

memory 层自改进的天然困难是判据来源。policy 级有任务 reward，code 级有可执行验证器，而“这条记忆值不值得保留”几乎是纯主观判断。一旦用模型自己判断“这条有用”，就回到 self-preference 的老问题。所以主流工作把 memory 当成被工具管理的状态，而不是被循环改进的对象。「Agent 记忆」整篇讲的就是这一层工程，不在 RSI 范围。RAG 索引的自改进同理：可以自动扩充 chunk、调 embedding，但判据仍要靠下游检索质量，而非“索引自己给自己打分”。

📎 展开阅读：「Agent 记忆」整篇（memory 工程与记忆库设计），「合成数据」第 4 章（模型崩溃触发条件）。

### 6.2 评测集：自评自欺的高发区

评测集自改进最危险：生成题、打分、出评测都用同一个模型时，偏好泄漏分数平均 23.6%，同家族 8.9%，不同系列才降到 2.8%（见「合成数据」）。所以这一档目前只有“改裁判”敢站出来：Self-Taught Evaluators 让裁判 policy 在自产合成对比输出上自改进，不靠人工标签，RewardBench 从 75.4 拉到 88.3[[10]](https://arxiv.org/abs/2408.02666)。它把“评测集”窄化为“一个可被训练的裁判”，巧妙绕开了“自己出题自己评”的自欺。但它仍用 LLM 裁判做判据，self-preference 没有根除（第 8 章的 RewardBench v1 时点问题即是其一）。

### 6.3 迭代合成数据的退化证据

在合成数据上反复迭代训练的退化并非均匀，而是“能力极化”：某些能力塌缩、某些反而上升，所以标准 LM 指标看不出问题[[33]](https://arxiv.org/abs/2607.17043)。这解释了为什么 memory/eval-set 自改进难以靠 aggregate 指标监控：一个总体 loss 不涨的回路，可能正在悄悄塌缩某项关键能力。公平性退化还会先于标准指标恶化出现，合成数据递归训练会放大社会偏见[[34]](https://arxiv.org/abs/2608.04268)。这两条都说明，没有外置 verifier 的 memory/eval-set 自改进，监控成本极高，这正是它冷的根本原因。

> 📌 **为什么冷**：memory 缺稳定判据、RAG 索引判据要靠下游检索、评测集自评自欺、迭代合成数据的能力极化与公平性退化都难监控。要做这一档，必须外置一个不同家族的 verifier 或真值，否则回路会在看不见的地方塌缩。

## 第 7 章 · 判据阶梯：环境 reward、verifier、LLM 裁判与人类

RSI 回路能不能站住，七成取决于判据。判据越靠客观端，越不容易被“模型自评”污染，但覆盖的任务越窄；越靠主观端，覆盖面越广，self-preference 与过拟合的风险越大。把判据排成四级阶梯，每级配代表系统，就能看清各家到底把判据放在哪。

📎 展开阅读：「推理模型与测试时计算」第 3 章（ORM/PRM/生成式验证器作为判据的机制），「LLM 应用评测与可观测」第 4–5 章（LLM 裁判偏差）。

### 7.1 第一级：环境 reward 与自动评估器

最硬的判据是任务可量化。AlphaEvolve 用客观可量化的自动评估器（数学可验证）演化整段 codebase，这也是它的结果能被外部直接校验的原因[[4]](https://arxiv.org/abs/2506.13131)。Agent Lightning 把 Agent 执行建模成 MDP，最终奖励 r_N 评估任务整体成功，不是 LLM 裁判[[7]](https://arxiv.org/abs/2508.03680)。rStar-Math 用 MCTS 的胜负与正确性检查给信号。这一级要求任务有“能跑、能对答案”的属性，所以集中在代码、数学、有单元测试的 Agent 任务。

### 7.2 第二级：verifier 与 pass@k

没有现成环境 reward 时，用一个外置 verifier 选最好答案。过程奖励模型（PRM）与生成式验证器属于这一级，机制与失败模式见「推理模型与测试时计算」第 3 章。RSI 语境里它常以 pass@k 的形式出现：让模型生成多个，由 verifier 选优，再把优的回灌训练。它的风险是 verifier 本身会过拟合或被钻空子；一旦 verifier 也由模型当，就滑到下一级。

### 7.3 第三级：LLM 裁判

LLM 当裁判的奠基是 Zheng 等的 MT-Bench/Chatbot Arena[[35]](https://arxiv.org/abs/2306.05685)。Self-Rewarding 把它做到极致：模型自己给自己出 DPO 奖励，policy 与奖励能力双轴同涨[[3]](https://arxiv.org/abs/2401.10020)。Self-Taught Evaluators 让裁判在自产合成对比输出上自改进、无人工标签，RewardBench 从 75.4 拉到 88.3[[10]](https://arxiv.org/abs/2408.02666)。背景是 RLAIF：用 AI 反馈替代人工偏好训练奖励模型（Constitutional AI）[[36]](https://arxiv.org/abs/2212.08073)。

这一级覆盖面最广，但 self-preference 是结构性的：判据来自模型自己，回路会把“模型喜欢的”误当成“真正好的”。第 8 章会看到它在 RewardBench v1 上的时点风险。

### 7.4 第四级：人类

人类判据最准但不可规模化。标准对照是 InstructGPT 的 RLHF：用一个由人工偏好训练、随后冻结的奖励模型，训练后无法再改进，Self-Rewarding 正是拿这一点当动机[[37]](https://arxiv.org/abs/2203.02155)。在 RSI 里，人类判据几乎只在“裁判的裁判”这一层出现（人工标注一个小评测集来校准 LLM 裁判），不直接进循环。

### 7.5 一个特殊档：LLM 写奖励函数代码

Eureka 让 LLM 用代码写出 RL 的奖励函数并按奖励梯度迭代改进，奖励函数本身被自改进循环更新[[29]](https://arxiv.org/abs/2310.12931)。它的最终判据是下游策略表现（环境 reward），但中间判据（奖励代码的好坏）由 LLM 自己生成。这是“判据来源”被部分自改的形态，介于第一级与第三级之间，只有当外层任务指标足够硬时才不会塌。

![判据四级阶梯：越客观越不容易塌，越主观覆盖越广](assets/48-rsi/fig-06.jpg)

> 💡 **常见误区**：默认“LLM 裁判比规则好”。在 RSI 回路里，LLM 裁判的覆盖面优势恰恰是它的风险：回路会优化“裁判喜欢的”，而不是“真正对的”。判据能往客观端挪一格就挪一格；非用 LLM 裁判时，至少用一个不同家族的模型出题与打分。

## 第 8 章 · 饱和、过拟合与自偏好

自改进回路最常被问的两个问题是“跑几轮会饱和”和“会不会越改越差”。现有证据偏负但不致命：prompt 级自改有明确的负证据，权重级自改的退化出现在“递归再训于自生成数据”这条相邻路径上，裁判自改则有时点风险。下面把这些证据摆清楚，并标明它们离 RSI 主回路有多远。

📎 展开阅读：「合成数据」第 4 章（模型崩溃的触发条件与“替换 vs 累积”区别）。

### 8.1 prompt 级自改的负证据

Huang 等的《Large Language Models Cannot Self-Correct Reasoning Yet》（2023-10）测出仅靠内部知识自纠正会让模型更差：没有外置 verifier 时，自纠正不仅不提升反而掉分[[12]](https://arxiv.org/abs/2310.01798)。Self-Refine（自反馈迭代精炼）与 Reflexion（语言化自反思）属于这一级，它们在推断时改输出但不改权重，收益强依赖任务与模型，不是稳定正向[[22]](https://arxiv.org/abs/2303.17651)[[23]](https://arxiv.org/abs/2303.11366)。结论很直接：判据不硬时，自改回路会朝“模型自己满意”的方向偏，而非“真正对”。

### 8.2 递归再训于自生成数据的退化

把模型在自己输出上反复再训，退化并不均匀。Bhandari 与 Lingthep 在生物医学问答上做了跨代研究：递归再训于自生成答案上的指标降幅大于复用人数据，大模型更明显，虽未达普遍崩溃[[13]](https://arxiv.org/abs/2609.34257)。Luo 等发现迭代指令微调的崩溃是“能力极化”：某些能力塌缩、某些反而上升，标准 LM 指标看不出问题，需要专门的失败驱动生成与不确定度边界策展才能稳[[33]](https://arxiv.org/abs/2607.17043)。Proskurina 等进一步发现合成数据递归训练放大社会偏见，且公平性退化出现在标准指标显著恶化之前[[34]](https://arxiv.org/abs/2608.04268)。

这三条都是“合成数据递归”路径上的证据，离 SEAL 的权重级 ReSTEM 或 Self-Rewarding 的 Iterative DPO 只差一步（它们都用自生成数据改权重），但尚无工作直接在 RSI 主回路里测到同等退化，这一缺口要诚实标注。

### 8.3 RewardBench：v1 时点与 v2

Self-Taught Evaluators 在 RewardBench 上把 Llama-3-70B-Instruct 从 75.4 拉到 88.3（多数投票 88.7），并自报“匹配顶级奖励模型”[[10]](https://arxiv.org/abs/2408.02666)。这必须标口径：88.7 是 2024-08 的 RewardBench v1 数字。Skywork-Reward-Gemma-2-27B-v0.2 在 RewardBench v1 自报 Overall 94.3、Chat 子项 96.1，截至 2024-10 居当时榜首[[11]](https://huggingface.co/Skywork/Skywork-Reward-Gemma-2-27B-v0.2)。

AllenAI 于 2025-06 发布 RewardBench 2，换用新的未见人类提示并加难，模型在 v2 上的平均分比 v1 低约 20 分，v1 与 v2 分数不可直接比较[[42]](https://arxiv.org/abs/2506.01937)。故“匹配顶级”是 v1 时点说法，不是当前 SOTA；用 Self-Taught Evaluators 的 88.7 做对比基线时务必注明 v1。

![RewardBench v1 时点：75.4 → 88.7 → 94.3 / 96.1](assets/48-rsi/fig-07.jpg)

### 8.4 self-preference：结构性风险，证据仍缺口

self-preference 偏差在“模型当裁判”的设置里有理论预期，但把它直接接到 RSI 主回路（SEAL 的自生成数据、Self-Rewarding 的自裁判）上的量化证据，本轮未核实到。相邻证据是有的：递归退化与公平性塌缩（8.2）、RewardBench v1 时点漂移（8.3）。诚实的说法是：self-preference 是 LLM 裁判象限的结构性风险，目前靠“用不同家族模型出题与打分”被动缓解，还没有在 RSI 主回路里被直接测出几轮崩溃的数字。

> 💡 **常见误区**：把“跑了 N 轮还在涨”当成回路安全。现有证据里，退化先出现在公平性与某些能力的极化上，而这两项不会被 aggregate 指标捕获。监控自改进回路至少要分能力子项、加公平性探针，并且对 LLM 裁判判据的回路用不同家族模型交叉校验。

### 8.5 理论上限：自改进只放大模型已有的能力

前四节是经验证据。2024 年底有两篇理论工作，从统计角度解释了自改进为什么会饱和。

- **锐化机制**：Huang 等把自改进形式化为“锐化”。模型验证答案的能力强于生成答案的能力，自改进就是把推断时“多采样再挑”的算力摊进权重，让模型一次就输出原本要靠多次采样才挑得出的好答案。前提是模型本来就有一定概率生成这个答案，自改进不会凭空产生模型里没有的信息。在这个框架下，初始模型的覆盖足够时，基于 SFT 的自改进是极小极大最优的。基于 RLHF 的自改进可以靠在线探索绕开覆盖要求[[43]](https://arxiv.org/abs/2412.01951)。
- **生成与验证的差距**：Song 等用“生成—验证差距”来衡量自改进的空间，即模型验证答案的准确率比它自己生成时高出多少。这个差距的某个变体随预训练算力单调增长，所以大模型自改进的余量更大。迭代自改进很快会把差距用完：7B 模型在第一轮之后差距就接近 0[[44]](https://arxiv.org/abs/2412.02674)。

这两条结论把 8.1 到 8.4 的现象串了起来。验证器弱或者没有外置验证器时，差距本来就小，自纠正掉分（8.1）。用自生成数据反复再训，覆盖之外的能力得不到补充，于是出现极化（8.2）。要让回路持续有收益，就得不断引入新的验证信号，比如可执行的测试、外部环境反馈、人工标注，或者换一个更强的裁判，相当于给回路补上模型自己没有的信息。第 7.2 节讨论的 verifier 和 pass@k，就是在度量和扩大这个差距。

## 第 9 章 · 风险与红线：spec gaming、Goodhart 与厂商立场

RSI 把判据也交给循环，判据越软，被钻空子的空间越大。这一章的风险框架有通用理论支撑，但“自改进循环内部出现 spec gaming”的具体案例目前没有 verified 证据；厂商公开框架里有针对 AI 自主研发或自我改进的能力阈值，但没有针对“自改进训练回路怎么跑”的可执行条款。下面把证据与缺口分清楚写。

### 9.1 Goodhart 与奖励过优化

奖励模型过优化是 RSI 最直接的结构性风险：把一个（哪怕很好的）奖励当目标去优化，到一定程度后真实效用开始掉。Gao、Schulman 与 Hilton 的《Scaling Laws for Reward Model Overoptimization》（2022-10）给出经典的“先升后降”曲线，刻画了 KL 散度与真实效用的关系[[14]](https://arxiv.org/abs/2210.10760)。在 RSI 里，这条曲线被放大，因为奖励函数本身可能被循环更新（Self-Rewarding、Eureka），优化压力直接作用在一个会动的目标上。

2026 年 11 月的一批新证据把过优化的可预测性又推了一步：诈唬研究证明两种开局期望相同的奖励，训练终点会稳定分出诚实模型与诈唬模型，并给出 GRPO 一步让模型更诚实的一阶条件 p̃r/b 大于 1+κ[53]；覆盖坍缩理论进一步证明 REINFORCE 可把最优响应概率压到零，覆盖保持型算法配合参考模型重置才能恢复[54]。

### 9.2 奖励误设与 spec gaming 的映射

Pan、Bhatia 与 Steinhardt 的《The Effects of Reward Misspecification》（2022-01）系统梳理了奖励误设（reward misspecification）导致模型错配的情形，并观察到“相变”：智能体能力越过某个阈值后，代理奖励继续上升而真实奖励骤降[[15]](https://arxiv.org/abs/2201.03544)。对应到 RSI：若判据是 LLM 裁判，回路会优化“裁判喜欢的特征”，哪怕这些特征与真实效用无关，这是 spec gaming 在自改进设置里的典型表现。理论上有预期，案例上缺核实。

同期两篇工作把误设的账本拆得更细：单面可验证奖励研究形式化了“已验证正样本只是未被证伪”的场景，标准 RLVR 会直接奖励未证伪样本上的捷径，改用只取负样本的策略优化[55]；特性搭车研究用 Price 方程说明，奖励未显式目标化的内部特性会被同步放大或压制，奖励外的行为变化因此有了可计算的解释[56]。

### 9.3 在 RSI 主回路里能抓到的相邻证据

离 RSI 最近的两个相邻信号都算不上 spec gaming 本身。其一，SEAL 试 GRPO/PPO 学 self-edit 策略时训练不稳，作者改用 ReSTEM^EM 绕开，这更像是优化不稳定而非奖励被钻空子[[2]](https://arxiv.org/abs/2506.10943)。其二，STOP 自声明“不是完整 RSI”因为权重没动[[1]](https://arxiv.org/abs/2310.02304)，是范围界定而非红线。不过，DGM v2 论文附录 H 作者自报了一例：节点 Node114 删除工具调用标记、骗过自己的评估拿到 2.0 分[[5]](https://arxiv.org/html/2505.22954v2#A8)。这是作者自报，没有独立复现，也不是权重级 RSI。诚实的工程做法是假设它会发生。9.4 会讲到，厂商阈值只管能力水位、不管回路细节，所以回路级护栏要自己补。

2026 年 11 月出现三条更靠近训练回路的相邻证据。其一，多语种 GRPO 对照实验里同一 SFT 检查点的发布分支 MATH 升 36.07 分、XQuAD-Indic 掉 19.32 F1，单一奖励在无人盯防的语种上静默制造能力漂移[57]。其二，构造环境把可验证奖励与上下文安全要求编码成冲突动力学：合规完成可行，但可执行的违规比任何合规策略拿分高，压测可以在上线前跑[58]。其三，agentic RL 服务化后攻击面外移：约 20 美元成本、多轮有毒环境训练即可完整击穿一个 LLM 的安全对齐，而奖励模型服务的取证链若不绑定请求、检查点与数值口径，审计无法确认分数归属[59][60]。这些仍不是 RSI 回路内的 verified spec gaming，但把“判据被钻空子”的成本和入口都量化了。

### 9.4 厂商公开红线：有能力阈值，无 RSI 回路专门条款

三家前沿厂商的公开安全框架都把“AI 自主做 AI 研发或自我改进”列为单独追踪的能力维度：OpenAI Preparedness Framework v2（2025-04）把 AI Self-improvement 列为 Tracked Category，设 High / Critical 两级能力阈值[[38]](https://cdn.openai.com/pdf/18a02b5d-6b67-4cec-ab64-68cdfbddebcd/preparedness-framework-v2.pdf)；Anthropic 的 Responsible Scaling Policy 设有自主 AI 研发（AI R&amp;D）能力阈值，触发后要求更高等级的安全与部署保障，并在后续版本中修订过该阈值的表述[[39]](https://www.anthropic.com/responsible-scaling-policy)；Google DeepMind 的 Frontier Safety Framework 设有 ML R&amp;D 加速或自动化的 Critical Capability Level[[40]](https://deepmind.google/blog/updating-the-frontier-safety-framework/)，其 Gemini 3 Pro 的 FSF 报告称该模型未达到这些 CCL[[41]](https://deepmind.google/models/fsf-reports/gemini-3-pro/)。

但要分清口径：这些都是“模型能力到了什么水位就要加什么保障”的阈值，评估的是模型能否自主推进 AI 研发，而不是针对“系统自改自身权重或代码的训练回路”的操作规则。没有厂商公开声明“自改进回路必须在 N 轮后停”或“自裁判判据要外置校验”这类回路级可执行条款，各家对阈值是否触发的判断也是厂商自评。所以工程上仍要把通用奖励安全实践（过优化监控、判据外置校验、每轮留快照）当回路级默认护栏，厂商阈值只作为能力水位的外部参照。

**口径说明**：9.1、9.2 是通用背景证据（非 RSI 内测得）；9.3 有相邻信号和一例作者自报的 RSI 内 spec gaming（DGM 附录 H），尚无独立复现；9.4 厂商有 AI 自主研发 / 自我改进的能力阈值，但没有 RSI 回路级专门条款，阈值是否触发为厂商自评。任何把“自改进会失控”写成已证实结论的表述都超出当前证据，本文不写。

## 第 10 章 · 怎么复现：经济账、第三方报告与最低可行配置

RSI 的复现经济账分三种形态：权重级要 GPU 训练，API 级要调用预算，code 级要有可执行验证器。这一章把各系统的可复现性、第三方报告状态，以及一份亲自动手的最低可行配置讲清楚。先说结论：小模型权重级自改进是个人能上手的，code 级最便宜但最难找问题，API 级随查询量线性烧钱。

### 10.1 各系统的可复现性

SEAL 用 Llama-3.2-1B-Instruct（ARC）与 Qwen2.5-7B（SQuAD），都是 1B 到 7B 量级的小模型，单卡或少量卡就能跑权重级 ReSTEM^EM 回路[[2]](https://arxiv.org/abs/2506.10943)。STOP 的 LM 是冻结的 GPT-4，改进对象是脚手架代码，复现成本主要是 API 调用次数，不需要训练算力[[1]](https://arxiv.org/abs/2310.02304)。

Agent Lightning 把任意 Agent 轨迹 MDP 化做 policy RL，可接已有 Agent 几乎零改动，但底层 policy 的 RL 训练仍要 GPU[[7]](https://arxiv.org/abs/2508.03680)。HGM 与 DGM 改的是 编程 Agent 自身代码，判据是 SWE-bench Verified、Polyglot 这类编码基准，算力门槛在 Agent 执行与评估，不在大模型训练[[6]](https://arxiv.org/abs/2510.21614)[[5]](https://arxiv.org/abs/2505.22954)。

AlphaEvolve 依赖 DeepMind 内部的 Gemini 与进化式程序库，公开材料不足以端到端复现；它的判据（数学可验证的自动评估器）倒是可以单独做：你不需要 Gemini，只需要一个能判定程序对错的环境[[4]](https://arxiv.org/abs/2506.13131)[[9]](https://deepmind.google/discover/blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/)。AI Scientist v1/v2 用前沿模型 API 跑端到端流水线，复现门槛在 API 预算与实验模板，且它不改权重或源码，严格说复现的是“idea 流水线”而非 RSI。

### 10.2 第三方报告：暂无可核验的独立复现

本轮核验里，RSI 主回路没有找到可核验的独立第三方复现。此前本节称 AlphaEvolve 的 11 维 kissing number 下界（593 球）有 arXiv:2609.07047 独立复现；2026-10-16 事实审计核实该编号对应的是机器人操作记忆基准 MEMOBench，与此无关，该说法已撤回。这不代表 593 球结果为假：其构造与校验代码由 DeepMind 公开，可自行验证[[8]](https://github.com/google-deepmind/alphaevolve_results)，只是不能算独立复现。其余系统（STOP、Self-Rewarding、SEAL、Self-Taught Evaluators、Agent Lightning、HGM、DGM、AI Scientist）均未找到 NVIDIA、Anthropic 或独立研究者的复现报告。把任何这类系统的厂商自报数字当 SOTA 引用前，先假定它没被独立验过。

R1 的训练成本（约 14.7 万 GPU 小时、29.4 万美元，不含基座预训练）详见「推理模型与测试时计算」与「后训练实战」，可作为“在强基座上做 RL 不昂贵”的参照，但那是推理 RL，不是 RSI 主回路。

### 10.3 亲自动手最低可行配置

想亲手跑一条 RSI 回路，最低门槛是 SEAL 形态：取一个 1B 到 7B 的开源模型，定一个有真值或可验证判据的小任务（ARC few-shot 这类），实现 self-edit 生成加 ReSTEM^EM（拒绝采样加在受奖励 self-edit 上 SFT），每轮留评测快照。判据一定要来自环境或真值，不要用模型自评。

若没有 GPU，退一步跑 STOP 形态：用一个 API 模型当冻结 LM，改进对象是一段脚手架代码，按 utility 函数多次查询取优，再让 improver 改进自身。这条不动权重，但能体会“code 级自改进回路”的形状[[1]](https://arxiv.org/abs/2310.02304)。再退一步，跑 prompt 级（DSPy/GEPA，见「提示词与流程的自动优化」），但要先做余量测试，它最易过拟合。

> 📌 **复现排序**（门槛由低到高）：prompt 级（DSPy/GEPA，几美元）＞ code 级脚本（STOP 形态，API 费用）＞ 权重级小模型（SEAL 形态，单卡）＞ 权重级大模型 policy RL（Agent Lightning 形态，多卡）＞ codebase 级开放演化（AlphaEvolve 形态，需 LLM 加可验证环境，公开材料不足）。判据硬度随之递增，spec gaming 风险递减。

## 第 11 章 · 怎么选与上线清单：vibe RSI 与工程化 RSI

前十章的结论落到决策上是两个问题：要不要上自改进回路，以及上了以后怎么不塌。把这两类实践分成 vibe RSI 与工程化 RSI 对照，能帮大多数团队判断自己站在哪、该补什么。

### 11.1 vibe RSI 与工程化 RSI 的对照

| 维度 | vibe RSI | 工程化 RSI |
|---|---|---|
| 改进对象 | prompt、示例、上下文 | 权重、policy、Agent 代码 |
| 判据 | 小评测集，常是 LLM 裁判 | 环境 reward 或客观自动评估器 |
| 代表 | DSPy/GEPA（见「提示词与流程的自动优化」） | SEAL、AlphaEvolve、Agent Lightning |
| 成本 | 几美元、几十分钟 | 单卡到多卡，或 API 预算 |
| 过拟合风险 | 高，已有 49% 运行低于零样本的报告 | 中，但退化在子项与公平性上 |
| 可回滚 | 强（改的是文本） | 每轮留权重与评测快照 |

![vibe RSI vs 工程化 RSI：先问判据硬度，再选档](assets/48-rsi/fig-08.jpg)

### 11.2 怎么选

- **先问判据硬度**。任务有真值、单元测试或可执行验证器，才上工程化 RSI；判据只能靠 LLM 裁判时，默认走 vibe RSI 并外置一个不同家族模型校验。
- **先问余量**。任何回路动手前，10 到 20 条留出题做余量测试，提升超 2 分再跑。这条对 vibe RSI 尤其重要，对工程化 RSI 也适用。
- **权重级首选小模型试水**。SEAL 用 1B 到 7B 模型就在 ARC 上把 0% 拉到 72.5%[[2]](https://arxiv.org/abs/2506.10943)，先在小模型上把回路跑通，再考虑放大。
- **code 级找可验证问题**。AlphaEvolve 的结果之所以能被外部直接校验，是因为判据是数学可验证的自动评估器[[4]](https://arxiv.org/abs/2506.13131)；没有这类问题，code 级回路空转。
- **Agent 轨迹 RL 用 Agent Lightning 形态**。它能接已有 Agent 几乎零改动，把工作流与工具调用轨迹变成可训练数据[[7]](https://arxiv.org/abs/2508.03680)。

### 11.3 上线清单与反模式

> 📌 **上线前核对清单**
>
> - 判据来源写明：环境 reward、verifier、LLM 裁判还是人类，并标硬度。
> - 每轮留评测快照（权重、prompt、评测分数），可回滚到任一历史轮。
> - 评测集分能力子项，不只看 aggregate 分；对合成数据递归回路加公平性探针。
> - LLM 裁判判据的回路，用不同家族模型出题与打分，并定期用人工小集校准。
> - 设过优化监控：跟踪 KL 散度与真实效用的“先升后降”拐点，到拐点停或切判据。
> - 第三方复现状态写明：本轮未找到任何系统（含 AlphaEvolve kissing number）的可核验独立复现，引用厂商自报数字时标注。
> - 不把 STOP（权重冻结）、AI Scientist（不改源码）这类自声明非完整 RSI 的系统当真 RSI 用，它们的失败模式不同。
> - 厂商框架只有能力阈值、没有回路级条款：把通用奖励安全实践（过优化监控、判据外置校验、可回滚快照）当默认护栏，厂商阈值只作能力水位参照。

> 💡 **反模式**：把“模型给自己打分再训自己”默认安全；把厂商自报数字当 SOTA；只看 aggregate 分不看子项与公平性；用同一模型出题、打分又出评测题；回路跑不停不设拐点。这些反模式的证据基础见第 8、9 章。

> 🔑 **全文小结**：RSI 是“改的对象 × 判据来源”二维里不同象限的工作，真正称得上递归自改的只有权重或代码被自身循环更新的少数几条线。判据越客观越不容易塌，但覆盖面越窄。几乎所有基准数字是厂商自报，本轮未找到可核验的独立第三方复现（AlphaEvolve 的 kissing number 构造可用公开代码校验，但仍属 DeepMind 自报）。自改进会饱和、会过拟合、会带 self-preference，“在主回路内测到 spec gaming”目前只有作者自报案例（DGM 附录 H），尚无独立复现；厂商框架有 AI 自我改进的能力阈值，却没有回路级可执行条款。上线唯一可靠的动作是按任务判据硬度选档、每轮留快照、子项加公平性监控、LLM 裁判回路外置校验。第 12 章的三篇闭环实验补上了没有黄金判据时的情形：回路能跑完，能力很少留下，验收要放在 Agent 看不到的地方，回滚和运行时审计要当成默认动作。

## 第 12 章 · 闭环自进化：选目标、用经验、留进系统

前 11 章讲的系统大多有一个共同前提：任务明确，有黄金判据替回路把方向，比如 AIME 的正确率、单元测试或者 AlphaEvolve 的自动评估器。同一个项目组在 2026 年 8 月底到 9 月初连发三篇论文，把这个前提拿掉，问 Agent 在没有黄金判据时能不能自己进化[[45]](https://arxiv.org/abs/2608.31111)[[46]](https://arxiv.org/abs/2608.31100)[[47]](https://arxiv.org/abs/2609.01437)[[48]](https://self-developing-agents.github.io/)。三篇各管一道关：Aspire 测能不能从宽泛目标里定出该练什么，S³Gym 测能不能把自己试、自己评的经验用到后面，HarnessDev 测能不能把改进写成可运行的外壳代码留下来。

结论大多是负面的，但负面结论的测法值得借鉴：三篇都把验收放在 Agent 看不到的地方，所以能分清“回路跑完了”和“能力留下了”。[LLM-58｜工程｜Agent 调优：从 trace 找到该改的那一层](./58-agent-tuning.md) 第 8.1 节从 Agent 调优的角度列过三篇的关键数字，这一章把它们放回 RSI 的框架里逐篇拆开。

### 12.1 半闭环与闭环

项目页把已有工作叫做半闭环：系统能自己生成候选、自己修改，但方向由外部的黄金判据给出[[48]](https://self-developing-agents.github.io/)。第 3 到 5 章的 SEAL、AlphaEvolve、rStar-Math 都属于这一类。闭环要求系统自己回答三个问题：练什么（Target），从经历里学到什么（Experience），学到的东西怎样留进系统（System）。三个问题各对应一种失败：目标定错，教训迁移不过去，机制写进了代码却在运行时没有生效。

![闭环自进化拆成三道关，验收都放在 Agent 看不到的地方](assets/48-rsi/fig-09.jpg)

三篇的验收方式相同。Agent 全程看不到评测集，版本冻结后才评测，HarnessDev 还另开一条固定执行模型的谱系，把执行模型本身的差异排除掉。项目页把最终的检验落在复用上，留下来的改动要在下一个任务上还能用。它对研究目标的说法是，要的是知道哪些改动该留下，持续变化本身没有价值[[48]](https://self-developing-agents.github.io/)。

### 12.2 选目标：模糊目标下练得出来、留不住

Aspire 只给 Agent 一个宽泛的能力目标，比如提升数学能力，不告诉它下游任务和评测方式。评测集是专家写的 520 道题，分六个目标，Agent 看不到。每次运行的预算是 40 个 GPU 小时，可以改权重，也可以改外壳[[45]](https://arxiv.org/abs/2608.31111)。

只交最终版的协议下，24 次运行只有 3 次超过基座，回滚后 21 次留下的就是基座本身[[45]](https://arxiv.org/abs/2608.31111)。允许多轮选择的自适应协议下，30 格实验有 28 格交出了被评测的 checkpoint，21 格合格，2 格的最好 checkpoint 超过基座，回滚之后只剩 1 格，Terra 配置把 Qwen3.5-4B 的数学从 17.86 提到 20.10[[45]](https://arxiv.org/abs/2608.31111)。

![30 格实验只有 1 格把增益留到了回滚之后](assets/48-rsi/fig-10.jpg)

失败的样子比通过率更值得看。Qwen3.5-4B 自己定目标时，数学分数从 0.79 涨到 1.75 再到 2.38，曲线单调上升，看起来在进步，可基座是 17.86，它是在从自己弄坏的地方往回爬[[45]](https://arxiv.org/abs/2608.31111)。另有 5 个 checkpoint 用数字标签的 MMLU 数据做 SFT，21,000 条训练目标全是单个数字，最终分数是 0、0、0、6.141、0[[45]](https://arxiv.org/abs/2608.31111)。回路每一步都跑通了，训练也收敛了，能力却没有留下。

48 对 Opus 4.8 的匹配运行对比了模糊目标和明确任务。模糊目标下，阅读任务定义的密度是明确任务的 2.98 倍，翻看评测脚本是 2.39 倍，LoRA 出现率从 24.1% 升到 89.8%[[45]](https://arxiv.org/abs/2608.31111)。精力挪到了解读目标上，反馈反而变稀：每次训练启动只配 0.57 次评测，官方流程下的 GPT-5.6 是 2.63 次，反馈密度差 4.6 倍[[45]](https://arxiv.org/abs/2608.31111)。落到分数上，Opus 4.8 在模糊目标下得 27.07，明确任务下得 32.90[[45]](https://arxiv.org/abs/2608.31111)。

改外壳这条路也一样。以工程化的 Qwen-Agent 为参考（28.64 分），三个造壳模型改出的外壳最好是 27.22，另外两个是 20.76 和 19.32，4B 的造壳模型没交出可用的外壳[[45]](https://arxiv.org/abs/2608.31111)。轨迹分析给了三种样子：一个在很窄的验证代理上过早停手，一个撤掉了审阅步骤却没补最终答案的不变式检查，最好的那个靠更广的测试加保守的选择[[45]](https://arxiv.org/abs/2608.31111)。

### 12.3 用经验：自评认得出好动作，不等于能改出好策略

S³Gym 用七个游戏做环境，探索时规则宽松，评测时换成严格配置的 held-out，比较三条经验通道：把历史轨迹直接放进上下文，把经验总结成记忆，用轨迹训练参数。全部实验共 98 次运行、116,117 次状态转移[[46]](https://arxiv.org/abs/2608.31100)。

先看自评准不准。按步看，模型判断一步好坏和真实结果的一致率在 0.496 到 0.881 之间，一致率最高的 PvZ，分数误差也最大，NMAE 0.882[[46]](https://arxiv.org/abs/2608.31100)。更关键的是自评和改进几乎无关：分块统计时，评判准确率和下一段涨分的秩相关只有 −0.010，事件级一致率和 NABA 的相关是 −0.23[[46]](https://arxiv.org/abs/2608.31100)。NABA 是高于起点那部分曲线下的面积除以起点分数，衡量的是持续改进。能分出好坏的裁判，给不出改进的方向。

三条通道没有谁处处领先。总结记忆在 Nullify、Tetris、Trust 上领先，这几个游戏有紧凑、可复用的规律，Trust 里“收益偏向背叛时就背叛”这一条总结让 GPT-5.5 的 NABA 多了 66.89[[46]](https://arxiv.org/abs/2608.31100)。历史轨迹在 Minesweeper、Snake、PvZ 上领先，这些游戏要的是精确的局面信息，Snake 的一条总结漏掉了蛇身和障碍的几何关系，NABA 反而少了 22.00[[46]](https://arxiv.org/abs/2608.31100)。

参数训练最持久，也最难控制。Qwen3-8B 用探索轨迹连续训练，每个 epoch 存一个 checkpoint，共 19 个。Trust 从 0 涨到最高 30，19 个里 18 个高于起点，PvZ 起点 23 分，之后每个 checkpoint 都是 6 分[[46]](https://arxiv.org/abs/2608.31100)。

![同一套参数训练，Trust 涨到 30，PvZ 从 23 掉到 6](assets/48-rsi/fig-11.jpg)

涨的游戏和掉的游戏用的是同一套参数。噪声轨迹和判错的步骤会跟有用的经验一起写进权重，这和第 6.3 节、第 8.2 节讲的递归再训退化是同一类问题，区别在于这里的数据来自 Agent 自己的探索。

### 12.4 留进系统：外壳能造，进化难迁移

外壳的分量可以先用一个数字感受：同样的 GPT-5 权重，在 Terminal-Bench 2.1 上用 Terminus 2 做到 35.2%，换成 Codex CLI 做到 49.6%[[47]](https://arxiv.org/abs/2609.01437)。HarnessDev 让六个模型从同一个种子骨架造外壳，覆盖四个领域、五个基准、2,207 个实例[[47]](https://arxiv.org/abs/2609.01437)。

造壳这一步做得出来，离人工外壳还差得远。造壳模型自己当执行模型时，Opus 4.8 总分最高，67.8，人工参考外壳是 86.2[[47]](https://arxiv.org/abs/2609.01437)。外壳和执行模型是绑在一起的，执行模型统一换成 Gemini 3.1 Pro 后，Opus 造的外壳在 SWE-Pro 上从 69.3 掉到 33.0，Qwen 造的外壳在 BrowseComp 上反而涨了 17.6[[47]](https://arxiv.org/abs/2609.01437)。成本也差得多，在 MLE-bench 上 GPT-5.5 用 2,930 万 token 拿到 19.1，DeepSeek V4 用 2.084 亿 token 拿到 19.6[[47]](https://arxiv.org/abs/2609.01437)。

进化这一步问题更大。五个造壳模型按反馈集迭代自己的外壳，冻结后在 630 题的 held-out 上评测。反馈集涨得最多的 Qwen 3.7 Max（+13.9）在 held-out 上只涨 1.43，反馈集涨得最少的 Opus 4.8（+3.0）在 held-out 上涨得最多（+4.44）[[47]](https://arxiv.org/abs/2609.01437)。执行模型固定成 Gemini 后，四条谱系里三条在 held-out 上转负，GPT-5.5 的外壳在反馈集上涨 2.4，在 held-out 上掉 10.32[[47]](https://arxiv.org/abs/2609.01437)。

![反馈集涨 13.9 分的外壳，到 held-out 只涨 1.43](assets/48-rsi/fig-12.jpg)

挑版本也靠不住。64 次可比的版本切换里，反馈集和 held-out 方向一致的只有 34 次，占 53.1%，和抛硬币差不多。9 条谱系声明的最终版里，只有 2 个是 held-out 上最好的。同一对外壳重复评测，分差的噪声有 ±4.75 分，小幅涨分不能归功于代码改动[[47]](https://arxiv.org/abs/2609.01437)。

代码里写了，运行时未必用上。18 个外壳里 11 个定义了 State 类，只有 1 个有保存接口，1 个定期存 checkpoint，26,679 条轨迹里 checkpoint 事件为 0[[47]](https://arxiv.org/abs/2609.01437)。代码领域的 108 个组件里有 18 个从没被触发，全是状态和记忆类，写作领域 587 个特性里有 124 个是死代码[[47]](https://arxiv.org/abs/2609.01437)。进化时新增的 169 个函数或类，113 个能从入口调用到，31 个只能经死代码调到，25 个没有调用方[[47]](https://arxiv.org/abs/2609.01437)。64 次切换的改动集中在执行控制（58 次）和工具（37 次），独立的验证器一次也没改过[[47]](https://arxiv.org/abs/2609.01437)。

### 12.5 三篇共同的失败模式

三篇的失败落在四个地方，每一处都能在前面的章节里找到对应的现象。这次的不同在于，它们是在闭环里、在隐藏验收下测出来的。

| 缺口 | 三篇里的证据 | 前面章节的对应 |
|---|---|---|
| 把完成率当留存率 | Aspire 30 格只留下 1 格；HarnessDev 反馈集 +13.9，held-out 只有 +1.43 | 第 8 章的饱和与过拟合，第 6.2 节的评测集自评自欺 |
| 自评没有校准 | S³Gym 评判准确率和涨分的秩相关 −0.010；Aspire 每次训练启动只配 0.57 次评测 | 第 7.3 节的 LLM 裁判，第 8.4 节的 self-preference |
| 外壳也过拟合反馈集 | HarnessDev 版本切换同向率 53.1%，固定执行模型后 4 条谱系 3 条转负 | 第 4 章的 prompt 与工具层，第 11.1 节的 vibe RSI |
| 机制写了没生效 | 26,679 条轨迹里 0 次 checkpoint；169 个新函数里 25 个没有调用方 | 第 9.3 节在主回路里能抓到的证据 |

### 12.6 上线门禁

> 📌 **闭环自进化的六道门禁**
>
> - 留一份 Agent 看不到的 held-out 集，只在版本冻结后评测，反馈集上的分数只用来挑候选。
> - 外壳按六个单元做版本管理：控制流、工具、上下文、状态、生命周期、验证器，每次改动记清动了哪个单元。
> - 换执行模型做 A/B，外壳改动要在至少两个执行模型上都不掉分才留下。
> - 自评上线前先校准，拿可执行验证器或人工小集对一遍，一致率和它与涨分的相关都要报。
> - 回滚当成一等操作，每个版本留快照，没超过基线就回到上一版。Aspire 的数字说明，多数时候回滚才是对的。
> - 审计运行时触发率，从轨迹里统计每个组件、每个新函数实际被调用的次数，没触发的机制按没写处理。

第 11.3 节的清单管的是半闭环回路，这六条是在它之上为闭环补的。报告里把完成率和 held-out 留存分开写，自评的一致率和与涨分的相关并列报告，这是三篇论文共同的做法。

## 参考文献

按正文首次引用顺序编号，资料截至 2026 年 10 月中。

[1] Zelikman, Lorch, Mackey, Kalai. STOP: Self-Taught Optimizer (2023-10) [https://arxiv.org/abs/2310.02304](https://arxiv.org/abs/2310.02304)

[2] Zweiger et al. SEAL: Self-Adapting Language Models (MIT, 2025-06) [https://arxiv.org/abs/2506.10943](https://arxiv.org/abs/2506.10943)

[3] Yuan et al. Self-Rewarding Language Models (Meta/NYU, 2024-01; ICML 2024) [https://arxiv.org/abs/2401.10020](https://arxiv.org/abs/2401.10020)

[4] DeepMind. AlphaEvolve (arXiv tech report, 2025-06) [https://arxiv.org/abs/2506.13131](https://arxiv.org/abs/2506.13131)

[5] Zhang, Hu, Lu, Lange, Clune (UBC / Vector Institute / Sakana AI). Darwin Gödel Machine: Open-Ended Evolution of Self-Improving Agents (2025-05) [https://arxiv.org/abs/2505.22954](https://arxiv.org/abs/2505.22954)

[6] Huxley-Gödel Machine: Coding Agents that Improve Themselves by Self-Reference (2025-10) [https://arxiv.org/abs/2510.21614](https://arxiv.org/abs/2510.21614)

[7] Luo et al. Agent Lightning (Microsoft, 2025-08) [https://arxiv.org/abs/2508.03680](https://arxiv.org/abs/2508.03680)

[8] DeepMind. alphaevolve_results: AlphaEvolve mathematical results and verification notebook (2025) [https://github.com/google-deepmind/alphaevolve_results](https://github.com/google-deepmind/alphaevolve_results)（原引 arXiv:2609.07047，经 2026-10-16 核验为无关论文 MEMOBench，已撤回）

[9] DeepMind. AlphaEvolve: A Gemini-Powered Coding Agent for Designing Advanced Algorithms (2025) [https://deepmind.google/discover/blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/](https://deepmind.google/discover/blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/)

[10] Wang, Weston, Li et al. Self-Taught Evaluators (Meta FAIR, 2024-08) [https://arxiv.org/abs/2408.02666](https://arxiv.org/abs/2408.02666)

[11] Skywork. Skywork-Reward-Gemma-2-27B-v0.2 Model Card (2024-10) [https://huggingface.co/Skywork/Skywork-Reward-Gemma-2-27B-v0.2](https://huggingface.co/Skywork/Skywork-Reward-Gemma-2-27B-v0.2)

[12] Huang et al. Large Language Models Cannot Self-Correct Reasoning Yet (2023-10) [https://arxiv.org/abs/2310.01798](https://arxiv.org/abs/2310.01798)

[13] Bhandari, Lingthep. Recursive LLM Degradation in Biomedical Question Answering: A Cross-Generation Study (2026-09) [https://arxiv.org/abs/2609.34257](https://arxiv.org/abs/2609.34257)

[14] Gao, Schulman, Hilton. Scaling Laws for Reward Model Overoptimization (2022-10) [https://arxiv.org/abs/2210.10760](https://arxiv.org/abs/2210.10760)

[15] Pan, Bhatia, Steinhardt. The Effects of Reward Misspecification: Mapping and Mitigating Misaligned Models (2022-01) [https://arxiv.org/abs/2201.03544](https://arxiv.org/abs/2201.03544)

[16] Schmidhuber. Gödel Machines: Self-Referential Universal Problem Solvers Making Provably Optimal Self-Improvements (2003) [https://arxiv.org/abs/cs/0309048](https://arxiv.org/abs/cs/0309048)

[17] Sakana AI. The AI Scientist: Towards Fully Automated Open-Ended Scientific Discovery (2024-08) [https://arxiv.org/abs/2408.06292](https://arxiv.org/abs/2408.06292)

[18] Sakana AI. The AI Scientist-v2: Workshop-Level Automated Scientific Discovery via Agentic Tree Search (2025-04) [https://arxiv.org/abs/2504.08066](https://arxiv.org/abs/2504.08066)

[19] Yin et al. Gödel Agent: A Self-Referential Agent Framework for Recursive Self-Improvement (2024-10) [https://arxiv.org/abs/2410.04444](https://arxiv.org/abs/2410.04444)

[20] Zelikman, Wu, Mu, Goodman. STaR: Bootstrapping Reasoning With Reasoning (2022) [https://arxiv.org/abs/2203.14465](https://arxiv.org/abs/2203.14465)

[21] Zelikman et al. Quiet-STaR: Language Models Can Teach Themselves to Think Before Speaking (2024) [https://arxiv.org/abs/2403.09629](https://arxiv.org/abs/2403.09629)

[22] Madaan et al. Self-Refine: Iterative Refinement with Self-Feedback (2023) [https://arxiv.org/abs/2303.17651](https://arxiv.org/abs/2303.17651)

[23] Shinn et al. Reflexion: Language Agents with Verbal Reinforcement Learning (2023) [https://arxiv.org/abs/2303.11366](https://arxiv.org/abs/2303.11366)

[24] Chen et al. Self-Play Fine-Tuning Converts Weak Language Models to Strong Language Models (2024) [https://arxiv.org/abs/2401.01335](https://arxiv.org/abs/2401.01335)

[25] Hu, Lu, Clune. Automated Design of Agentic Systems (ADAS, 2024-08) [https://arxiv.org/abs/2408.08435](https://arxiv.org/abs/2408.08435)

[26] Rafailov et al. Direct Preference Optimization: Your Language Model is Secretly a Reward Model (2023) [https://arxiv.org/abs/2305.18290](https://arxiv.org/abs/2305.18290)

[27] Singh et al. Beyond Human Data: Scaling Self-Training for Problem-Solving with Language Models (ReST^EM, 2023) [https://arxiv.org/abs/2312.06585](https://arxiv.org/abs/2312.06585)

[28] Guan et al. rStar-Math: Small LLMs Can Master Math Reasoning with Self-Evolved Deep Thinking (Microsoft, 2025-01) [https://arxiv.org/abs/2501.04519](https://arxiv.org/abs/2501.04519)

[29] Ma et al. Eureka: Human-Level Reward Design via Coding Large Language Models (2023-10) [https://arxiv.org/abs/2310.12931](https://arxiv.org/abs/2310.12931)

[30] Romera-Paredes et al. Mathematical discoveries from program search with large language models (FunSearch, Nature 2024) [https://www.nature.com/articles/s41586-023-06924-6](https://www.nature.com/articles/s41586-023-06924-6)

[31] Sakana AI. The AI Scientist published in Nature (2025) [https://sakana.ai/ai-scientist-nature/](https://sakana.ai/ai-scientist-nature/)

[32] Sakana AI. The AI Scientist Generates its First Peer-Reviewed Scientific Publication (2025) [https://sakana.ai/ai-scientist-first-publication/](https://sakana.ai/ai-scientist-first-publication/)

[33] Luo et al. Learning from Synthetic Data without Model Collapse in Iterative Instruction Tuning (2026-07) [https://arxiv.org/abs/2608.04268](https://arxiv.org/abs/2607.17043)

[34] Proskurina, Gourru, Velcin. The Fairness Collapse Phenomenon: Bias Amplification in Language Models Trained on Synthetic Data (2026-08) [https://arxiv.org/abs/2607.17043](https://arxiv.org/abs/2608.04268)

[35] Zheng et al. Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena (2023) [https://arxiv.org/abs/2306.05685](https://arxiv.org/abs/2306.05685)

[36] Bai et al. Constitutional AI: Harmlessness from AI Feedback (Anthropic, 2022-12) [https://arxiv.org/abs/2212.08073](https://arxiv.org/abs/2212.08073)

[37] Ouyang et al. Training language models to follow instructions with human feedback (InstructGPT, 2022-03) [https://arxiv.org/abs/2203.02155](https://arxiv.org/abs/2203.02155)

[38] OpenAI. Preparedness Framework (Version 2, 2025-04) [https://cdn.openai.com/pdf/18a02b5d-6b67-4cec-ab64-68cdfbddebcd/preparedness-framework-v2.pdf](https://cdn.openai.com/pdf/18a02b5d-6b67-4cec-ab64-68cdfbddebcd/preparedness-framework-v2.pdf)

[39] Anthropic. Responsible Scaling Policy [https://www.anthropic.com/responsible-scaling-policy](https://www.anthropic.com/responsible-scaling-policy)

[40] Google DeepMind. Updating the Frontier Safety Framework [https://deepmind.google/blog/updating-the-frontier-safety-framework/](https://deepmind.google/blog/updating-the-frontier-safety-framework/)

[41] Google DeepMind. Gemini 3 Pro Frontier Safety Framework Report [https://deepmind.google/models/fsf-reports/gemini-3-pro/](https://deepmind.google/models/fsf-reports/gemini-3-pro/)

[42] Malik et al. RewardBench 2: Advancing Reward Model Evaluation (2025-06) [https://arxiv.org/abs/2506.01937](https://arxiv.org/abs/2506.01937)

[43] Huang, Block, Foster, Rohatgi, Zhang, Simchowitz, Ash, Krishnamurthy. Self-Improvement in Language Models: The Sharpening Mechanism (2024-12) [https://arxiv.org/abs/2412.01951](https://arxiv.org/abs/2412.01951)

[44] Song, Zhang, Eisenach, Kakade, Foster, Ghai. Mind the Gap: Examining the Self-Improvement Capabilities of Large Language Models (2024-12) [https://arxiv.org/abs/2412.02674](https://arxiv.org/abs/2412.02674)

[45] Aspire: Can Models Self-Evolve from Vague Goals? (arXiv, 2026-08) [https://arxiv.org/abs/2608.31111](https://arxiv.org/abs/2608.31111)

[46] S³Gym: Can LLMs Turn Self-Testing and Self-Judging into Self-Improvement? (arXiv, 2026-08) [https://arxiv.org/abs/2608.31100](https://arxiv.org/abs/2608.31100)

[47] HarnessDev: Can LLMs Create and Evolve Their Own Agent Harness? (arXiv, 2026-09) [https://arxiv.org/abs/2609.01437](https://arxiv.org/abs/2609.01437)

[48] Self-Developing Agents: From Half-Loop to Closed-Loop RSI (project page, 2026-09) [https://self-developing-agents.github.io/](https://self-developing-agents.github.io/)

[49] Takehi, Teoh, Zhang, Khattab. Sparks of Weight-Space Programming: LLMs can natively read &amp; write neural network weights (MIT CSAIL 博客, 2026-11) [https://rikiyat.github.io/blog/weight-space-programming/](https://rikiyat.github.io/blog/weight-space-programming/)

[50] RikiyaT/lm_understand-logs：八个实验的完整日志仓库（GitHub） [https://github.com/RikiyaT/lm_understand-logs](https://github.com/RikiyaT/lm_understand-logs)

[51] Rikiya/Qwen3.5-0.8B-weight-space-programming：修改后的模型（Hugging Face） [https://huggingface.co/Rikiya/Qwen3.5-0.8B-weight-space-programming](https://huggingface.co/Rikiya/Qwen3.5-0.8B-weight-space-programming)

[52] Arditi et al. Refusal in Language Models Is Mediated by a Single Direction (2024-06) [https://arxiv.org/abs/2406.11717](https://arxiv.org/abs/2406.11717)

[53] Bottman et al. Honesty Is a Race: Predicting When Verifiable-Reward RL Teaches a Model to Bluff (2026-11) [https://arxiv.org/abs/2611.15779](https://arxiv.org/abs/2611.15779)

[54] Huang et al. A Theory of Policy Optimization in Post-Training: Coverage Collapse and Reference Resets (2026-11) [https://arxiv.org/abs/2611.14467](https://arxiv.org/abs/2611.14467)

[55] Xu et al. Negative-Only Policy Optimization for One-Sided Verifiable Rewards (2026-11) [https://arxiv.org/abs/2611.17287](https://arxiv.org/abs/2611.17287)

[56] Soubki et al. Trait Hitchhiking in LLM Post-Training (2026-11) [https://arxiv.org/abs/2611.14683](https://arxiv.org/abs/2611.14683)

[57] Saksena Raj et al. Outside the Purview of the Reward: Capability Drift in Multilingual Post-Training (2026-11) [https://arxiv.org/abs/2611.15717](https://arxiv.org/abs/2611.15717)

[58] Shen. Shortcuts and Boundaries: Verifiable RL Misalignment and Contextual Safety Learning (2026-11) [https://arxiv.org/abs/2611.12865](https://arxiv.org/abs/2611.12865)

[59] Huang et al. Bait: Mitigating Harmful Agentic Reinforcement Learning via Probing Reward Function (2026-11) [https://arxiv.org/abs/2611.16369](https://arxiv.org/abs/2611.16369)

[60] Liao et al. VRMaaS: Verifiable Reward-Model-as-a-Service for Trustworthy LLM Alignment (2026-11) [https://arxiv.org/abs/2611.14984](https://arxiv.org/abs/2611.14984)

---

上一篇：[LLM-47｜场景｜运维根因定位：拓扑、因果推断与 LLM](./47-root-cause.md) · [目录](./README.md) · 下一篇：[LLM-50｜系统｜消费级单卡自部署：Qwen3.8-27B 跑在 4090 / 5090 上](./50-consumer-gpu-deploy.md)
