# CheckerBench：让 Agent 从零写一个静态分析检查器，最强组合也只过了 45%

上周刷到一篇 benchmark 论文（arXiv 2610.07557），看完第一反应是：终于有人把"让 Agent 写静态分析 checker"这件事拉出来单独拷打了。

我们平时测 coding agent，基本就是 SWE-bench 那一套——给个 issue，让它改 bug，跑测试看 patch 对不对。但安全团队的真实工作流里还有一件更硬核的事：**写静态分析检查器**。就是给 Clang Static Analyzer 或者 CodeQL 写一条规则，让它能在整个仓库里自动找出某一类漏洞。这件事比修单个 bug 难得多：你得读懂缺陷机理、搞懂分析器框架的 API、写出能编译的 checker 代码、还要控制误报。这活儿以前都是安全专家手工干的。

那么问题来了——现在的 coding agent 到底能不能干这活儿？CheckerBench 给出的答案不太乐观：21 个模型×框架组合跑下来，平均 Pass@1 只有 32.30%，最强的 Claude Opus 4.8 + OpenCode 也就 45.33%。

**核心摘要**：这篇论文构建了 CheckerBench——首个面向"长程静态分析 checker 合成"的可执行基准，包含 300 个任务，来自 297 个真实 CVE、167 个开源仓库、85 种 CWE 缺陷类型和 5 种语言生态。配套的 CheckerLab 评估框架会独立重建 agent 提交的 checker，检查它在漏洞版/修复版代码上的诊断对比度、补丁定位能力和主线误报率。实验覆盖 7 个模型 × 3 种 agent harness、每个配置独立重复 3 次。我的判断：这不是一篇刷榜论文，而是一篇把"agent 到底行不行"量化出来的压力测试，数据扎实，失败分析尤其有料，值得做 agent 和安全工具的人都看一眼。

---

## 📖 论文信息

- **标题**：CheckerBench: Can Long-Horizon Agents Synthesize Static-Analysis Checkers?
- **作者**：Hang He, Li Wang, Hao Chen, Yuchen Shao, Yuling Shi, Lisheng Wang, Peiyang Liu, Goose Lin, Zaiyuan Wang, Haiying Sun, Ting Su, Chengcheng Wan
- **机构**：East China Normal University、Humanlaya Data、Shanghai Jiao Tong University、Peking University、Shanghai Innovation Institute
- **链接**：https://arxiv.org/abs/2610.07557
- **代码/数据**：GitHub（ahang0712/CheckerBench-Dataset）、Hugging Face（Benchanything/checkerbench）

---

## 🎯 现有 benchmark 到底差在哪

作者把现有评测的短板归纳成四个"反复出现的评估缺口"，这张图讲得挺清楚的：

![图2：四类反复出现的评估缺口](https://arxiv.org/html/2610.07557v1/figure1.png)

*图2：现有 benchmark 的四个缺口——(a) 不以 checker 为中心，patch 修一个 bug 不等于造出可复用的检查器；(b) 交互视野有限，预定义阶段的工作流锁死了迭代空间；(c) scaffold 各自为政，同一个 LLM 换套框架分数就变；(d) 正确性信号碎片化，patch 通过测试不能证明 checker 是对的。*

说实话，这四条里我觉得最扎心的是 (d)。SWE-bench 式评测的信号是"测试通过/不通过"，但 checker 的正确性完全是另一回事——它得在**漏洞版本上报警、在修复版本上闭嘴**，还得在仓库主线代码上别把误报刷爆。这种"诊断对比度"信号，现有 benchmark 根本没有。

还有一个背景值得顺手掏出来：checker 合成这个方向其实已有一些专门系统，比如 KNighter（从 Linux 内核补丁合成 CSA checker）、QLCoder（迭代生成 CodeQL 查询）、IRIS（推断污点分析规约）。但这些系统各自绑死特定分析器和特定领域，互相之间没法比。CheckerBench 想做的是给这个方向立一把统一的尺子。

---

## 🏗️ CheckerBench 是怎么造出来的

任务的形式化定义很直接：每个任务 $x_i = (p_i, \ell_i, \alpha_i, \mathcal{E}_i, \mathcal{Q}_i)$，包含一个漏洞-修复对 $p_i = (r_i, c_i^-, c_i^+, \Delta_i)$（仓库、漏洞版本、修复版本、补丁）、语言、分析器、版本钉死的可复现环境和合成指令。agent 在时间预算 $B$ 内产出轨迹和候选 checker，而一个**在 rollout 之前就冻结的隐藏验证器** $\mathcal{V}_i$ 负责最终裁决。

![图3：CheckerBench 的六阶段构建流水线与真实任务示例](https://arxiv.org/html/2610.07557v1/figure3-construction-example.png)

*图3：(a) 六阶段构建流水线；(b) 一个真实任务示例——FFmpeg 的 PNG 缓冲区溢出（CVE-2017-7866）。左侧是漏洞版代码（缓冲区写满后 NUL 终止符越界），中间是修复版（预留一字节），右侧是生成的 CSA checker 片段及验证结果：漏洞版报 1 个警告、修复版报 0 个，归档通过用了 41 步交互。*

六个阶段拆开看：

1. **漏洞源收集**：从 NVD 把 CVE 记录链接到开源仓库的修复 commit，要求能取到漏洞版/修复版配对，且补丁暴露出可复用的静态分析性质。
2. **可执行环境重建**：钉死工具链和分析器——C/C++ 用 Clang Static Analyzer（CSA），其余四种语言用 CodeQL，恢复编译数据库和可分析范围。重建不出来的直接丢弃。
3. **任务打包**：给出补丁、修复前后函数、构建元数据、编译/扫描命令和一个最小 checker 脚手架。工作区 ID 随机化、CVE/CWE 标签隐藏，防止 agent 走"认出这个 CVE 然后背答案"的捷径。
4. **独立验证器构建**：rollout 之前冻结，在干净环境里从源码重建 checker 并跑差分验证。
5. **人工引导 rollout 采样**：校准阶段允许人工诊断失败、改 prompt 重跑（每个任务最多 3 次尝试），但只有 checker 通过冻结验证器**且轨迹至少 40 步交互**的才进样本池——保证任务确实是长程的。
6. **质控去重切分**：去除重复版本对，把同仓库、同 CVE、相似补丁的任务锁在同一个 split 里防泄漏。

最终数据集的分布长这样：

![图4：语言、仓库、CWE 分布](https://arxiv.org/html/2610.07557v1/figure3.png)

*图4：(a) 语言分布：C/C++ 159 个任务（53%）占大头，Python 51 个，Java/JavaScript/Go 各 30 个；(b) 仓库分布：Linux 内核一家就贡献 107 个任务（35.7%），FFmpeg、ImageMagick、JerryScript、QEMU 各 4 个；(c) CWE 分布：CWE-787 越界写（26 个）、CWE-125 越界读（21 个）、CWE-22 路径遍历（15 个）最多，尾部 227 个任务铺在 72.3% 的长尾 CWE 上。*

这个分布我觉得设计得还算良心——没有被某几个常见 CWE 刷满，长尾缺陷占七成多，想靠背 pattern 蒙混过关不太现实。

---

## 🔬 CheckerLab：checker 交上来之后怎么判

这部分是整篇论文我最想细讲的。很多 agent benchmark 的评分环节其实挺糊弄，CheckerLab 的做法是把"合成"和"评估"彻底分开：agent 在 50 分钟预算内干活、提交后源码立刻冻结，然后一个独立验证器在干净环境里从零重建 checker，跑完整套关卡。

![图5：CheckerLab 工作流与 TorchServe 试点轨迹](https://arxiv.org/html/2610.07557v1/figure5.png)

*图5：(a) CheckerLab 流程——agent 可见的是任务包（补丁、版本化代码、分析器、技能库），提交后才进入独立验证：重建、漏洞/修复版诊断对比、补丁相关性、主线误报扫描（最多抽 10 条告警人工复核级审查）。门槛表：CSA 要求 Process ≥80/100 且 Semantic ≥7/10，CodeQL 要求 Semantic ≥8/10，两者 FP 率都要 ≤25%。(b) 一个归档试点：TorchServe 的 Zip Slip 漏洞（CVE-2023-48299），agent 经历了"读补丁→实现 ZipSlipFlow 查询→诊断 TAR 流覆盖不足→去掉 sanitizer barrier 做对照→定稿"五个阶段，56 轮对话、71 次工具调用后通过验证。*

指标体系里几个关键定义：

- **Pass@1**：主指标，单次独立合成后 checker 通过全部关卡的百分比。每个配置在同样 300 个任务上独立重复 3 次取平均。生成失败、拒答、候选代码报错一律算失败——这条挺狠但也合理。
- **Build / Diff. SR**：独立编译通过率 / 差分验证通过率（漏洞版报警、修复版消失）。
- **Process / Semantic**：由 Claude Opus 5 当裁判打的过程质量分（百分制）和漏洞语义分（十分制），人工验证显示裁判与人类标签的 Cohen's $\kappa = 0.85$，一致性相当高。
- **FP 率**：$\mathrm{FP}_{\mathrm{pass}} = 100 \sum \mathrm{FP} / \sum (\mathrm{TP} + \mathrm{FP})$，统计通过 checker 在主线扫描中的告警，默认 25% 为录取线。

注意那个设计细节：验证器、参考 checker、校准轨迹对 agent 全部隐藏，而且 CSA 任务要求"消失的告警必须落在补丁改过的函数里"——防止 agent 写一个随便在哪报警的 checker 碰运气。

---

## 📊 实验：21 个组合的大摸底

实验阵容是 7 个模型（GPT-5.6、Claude Opus 4.8、Qwen3.8-Max、DeepSeek-V4-Pro、GLM-5.2、Kimi-K3、HY4-Preview）× 3 个 harness（OpenCode、Claude Code、Hermes Agent），每个跑在独立 E2B 沙箱里，50 分钟预算。

![图1：21 个模型-harness 组合的 Pass@1](https://arxiv.org/html/2610.07557v1/table3-pass-at-1-paper.png)

*图1：21 个组合的 Pass@1 柱状图。最高是 Claude Opus 4.8 + OpenCode 的 45.33%，最低是 HY4-Preview + Hermes Agent 的 21.67%。红色为 OpenCode，青色为 Claude Code，紫色为 Hermes Agent。*

主表（Table 2）数据量很大，我把关键列整理成下面这张表（数值均为三次重复均值）：

| 模型 | OpenCode Pass@1 | Claude Code Pass@1 | Hermes Agent Pass@1 |
|---|---|---|---|
| GPT-5.6 | 31.67 | 34.00 | 23.00 |
| Claude Opus 4.8 | **45.33** | 35.00 | 34.33 |
| Qwen3.8-Max | 37.33 | 34.67 | 25.00 |
| DeepSeek-V4-Pro | 39.67 | 38.33 | 32.33 |
| GLM-5.2 | 32.33 | 33.00 | 22.33 |
| Kimi-K3 | 38.67 | 38.00 | 24.33 |
| HY4-Preview | 33.67 | 23.67 | 21.67 |
| **Harness 平均** | **36.95** | 33.81 | 26.14 |

*表：7 模型 × 3 harness 的 Pass@1（%）。Overall Average 为 32.30%。*

几个让我停下来想了一下的发现：

**Build 和 Pass@1 之间的鸿沟比想象中大得多。** 整体 Build 率 78.25%，Diff. SR 73.65%，但 Pass@1 只有 32.30%——中间差了 41.35 个点。极端案例是 GPT-5.6 + Hermes Agent：Build 100%，Diff. SR 64%，Pass@1 却只有 23%。也就是说，"能编译、能在漏洞版上报警"和"语义对、过程规范、误报可控"之间隔着一条大河。你要是只看编译通过率去评估这个任务，会得出完全错误的结论。

**Harness 的影响大到会改写模型排名。** OpenCode 平均比 Hermes Agent 高 10.81 个点。Claude Opus 4.8 在 OpenCode 和 Hermes 上排第一，但换到 Claude Code 上，DeepSeek-V4-Pro 以 38.33% 反超它的 35.00%。Kimi-K3 更夸张，从 OpenCode 的 38.67% 掉到 Hermes 的 24.33%，差了 14.34 个点。所以以后比 agent 能力，"模型 + harness"得当成一个整体单元来看，单报模型分数意义不大。

**质量分和成功率居然会脱钩。** Qwen3.8-Max + Hermes 拿下了全场最高的 Process（90.92/100）和 Semantic（8.53/10），但 Pass@1 只有 25%，Build 才 44%。写得一手"漂亮但编译不过"的 checker——这个组合把过程规范学得挺好，基本功却拉了。反过来 GPT-5.6 + OpenCode 的 FP 率 0% 但只报了 7 条主线警告，而 Claude Opus 4.8 报了 174 条、FP 率 13.79%。**没有任何配置能同时领先成功率、质量和精度**，三者必须一起看。

**C/C++ 是共同的噩梦。** 每个模型在 C/C++ 上的 Pass@1 都是各科最低，最强的 DeepSeek-V4-Pro 也只有 24.53%。而 C/C++ 占 300 个任务里的 159 个，直接把总平均分压下去了。其他语言的领头羊各不相同：Python 是 Claude Opus 4.8（65.36%），Go 是 DeepSeek-V4-Pro（53.33%），Java 和 JavaScript 都是 Qwen3.8-Max（54.44% / 53.33%）。

![图6：按语言、CWE 细分的 Pass@1 及效率散点](https://arxiv.org/html/2610.07557v1/model-performance-analysis.png)

*图6：(a)(b) 语言和 CWE 维度的 Pass@1 热力图，C/C++ 一列明显偏冷；(c) 成功轨迹的平均轮次 vs Pass@1 散点——Kimi-K3 在三个 harness 里都是成功轨迹上工具调用最少的，在 Claude Code 里以 31.81 轮、38.61 次工具调用拿到 38.00%，比 DeepSeek-V4-Pro 的 47.17 轮、62.50 次调用省了 38.2%，Pass@1 只差 0.33 个点。*

效率这个维度值得单独说。如果你要拿这套东西做生产工具，成本敏感度会很高——Kimi-K3 这种"分数不顶尖但活儿干得省"的配置，工程上可能比分数第一的配置更划算。

---

## 🧪 对比、消融与敏感性

**和 KNighter 的正面对比**（Table 3）：在 KNighter 自己的 61 个 Linux 内核任务上，用 DeepSeek-V4-Pro + OpenCode 实现论文的技能引导式合成方法，有效 checker 率 80.3%（49/61），KNighter 本身是 67.2%（41/61）。去掉共享技能库后直接崩到 29.5%。这个对比说明两件事：技能库这套东西确实有用，而且它能迁移到别人定义的任务集上，不是过拟合自家 benchmark。

**消融实验**（Table 4，DeepSeek-V4-Pro）把各组件的贡献扒得很清楚：

| 配置 | Pass@1 (%) | Build (%) | Diff. SR (%) |
|---|---|---|---|
| Full（完整） | 39.67 | 92.67 | 90.67 |
| 去掉技能库 | 8.33 | 79.33 | 52.67 |
| 隐藏执行反馈 | 13.33 | 67.33 | 41.67 |
| 无 harness 单发生成 | 0.33 | 3.33 | 3.33 |

单发生成（one-shot、无工具无 harness）只有 0.33%——300 个任务里 237 个交上来的都是无效或不完整的东西。这个数字基本宣告了"直接让 LLM 写 checker"这条路在这个任务上彻底行不通，迭代式 agent 工作流是刚需，不是加分项。

**时间预算敏感性**：对已有轨迹回溯性地施加 10/20/30/40/50 分钟截断，Pass@1 从 10 分钟的 7.24% 爬到 30 分钟的 26.05%、50 分钟的 32.30%。40 分钟截断已经能保住 93.02% 的成功案例，最后 10 分钟只贡献 2.25 个点，边际收益明显递减。

![图8：三种 harness 的 Pass@1 随时间预算变化](https://arxiv.org/html/2610.07557v1/time-budget-sensitivity.png)

*图8：Pass@1 vs 活跃生成时间。三条曲线都在 30 分钟后趋于平缓，OpenCode 全程领先。*

**FP 阈值敏感性**：把误报录取线从 10% 放宽到 50%，总通过率只从 32.05% 涨到 32.83%，净增 0.78 个点——结论对阈值选择很稳健，不是靠调阈值刷出来的。

---

## 🔍 失败分析：4360 条失败轨迹在说什么

这部分是全文我觉得最值钱的地方。作者翻了 4360 条失败运行，归类出失败原因：

| 失败类别 | 占比 |
|---|---|
| 预算耗尽（超时没干完） | 25.32% |
| 合成过程不充分 | 22.96% |
| 语义建模不到位 | 21.15% |
| 误报过多 | 15.96% |
| 差分检测/定位失败 | 8.17% |
| 补丁过拟合/锚定 | 3.56% |
| 产物缺失或无效 | 1.83% |
| 编译/注册失败 | 1.05% |

前三大原因加起来快七成了。论文里举的一个例子特别典型：agent 计划里写的是"路径敏感分析"，实现出来却退化成纯 AST 模式匹配，把锁条件和控制流全丢了。**计划和实现之间的落差**——这其实是当前 agent 的通病，只是在 checker 合成这种对语义精度要求极高的任务上被放大得特别明显。

还有 3.56% 的"补丁锚定"失败：agent 盯着补丁本身写出一个只对这一个补丁有效的 checker，而不是抽象出可复用的缺陷模式。这恰好是 checker 合成和普通 bug 修复最本质的区别。

**数据污染审计**也做了：作者对 100 个语言分层任务做了原文/改写/标识符替换三组对照（O/P/D），Claude Opus 4.8 和 DeepSeek-V4-Pro 的精确 CVE 召回从 10%→1%、8%→1%，掉了 7-9 个点，但 Pass@1 只掉了 2-3 个点。换句话说，"认出这个 CVE"带来的收益有限——任务设计上把记忆捷径的收益压得很低，测的主要是真本事。

---

## 🤔 我的判断

先说亮点。

这个 benchmark 的工程质量在同类工作里算上乘：冻结验证器、隐藏参考解、版本钉死的环境、三次独立重复、裁判模型过人工一致性验证（κ=0.85）、污染审计——该做的防护基本都做了。对比 Table 1 里那一排"部分支持"的前作，CheckerBench 是唯一把仓库上下文、可执行环境、多 harness、长程交互、checker 合成、语义验证、误报评估七项全打勾的。

失败分析那 4360 条轨迹是真正的宝藏数据。它告诉你当前 agent 卡在哪：不是不会写代码，而是规划与实现脱节、语义建模不精确、以及管不住误报。做 agent 改进的人可以直接拿这个当 roadmap。

再说问题。

一个是 C/C++ 占比 53% 且全是 CSA 任务，而 CSA 的开发体验（回调注册、ProgramState 操作）和 CodeQL 的声明式查询差别巨大，总平均分在很大程度上其实是在度量"CSA checker 有多难写"。分语言看结论会更诚实，论文确实也给了，但"32.30% 平均分"这个传播最广的数字掩盖了结构差异。

另一个是 50 分钟预算。真实安全工程师写一个生产级 checker 往往以天计，50 分钟卡死之后，25.32% 的失败是"预算耗尽"——这里面有多少是"再给一小时就能做完"，从回溯截断实验看边际收益递减，但"在更短截止下重跑"这件事论文明确说没测过，所以预算和难度的纠缠没有完全拆开。

还有就是 LLM 裁判。κ=0.85 确实不错，但 Process/Semantic 打分终究有裁判模型自己的偏好在里面，换一家裁判会不会改排名，没做敏感性测试。

瑕不掩瑜。我的总体评价：**这是一篇定位清晰、执行扎实的 benchmark 论文**，它最大的贡献不是某个数字，而是把"agent 能不能造安全工具"这件事从闲聊变成了可测量、可复现、可对比的实验科学。32.30% 这个分数也给整个行业泼了盆冷水——离"agent 自动帮你写 CodeQL 查询库"的愿景，还差得远。

---

## 💡 工程启发

如果你在做安全工具或 agent 产品，有几个可以直接拿走的东西：

- **技能库 + 执行反馈是刚需**：消融里去掉技能库从 39.67% 掉到 8.33%，去掉执行反馈掉到 13.33%。给 agent 配 analyzer 文档和可复用 checker 示例，比换更强的模型可能收益更大。
- **评估 agent 别只看编译和通过率**：Build 78% 而 Pass@1 32% 的鸿沟提醒我们，"看起来在干活"和"活儿干对了"之间要加语义和误报关卡。
- **选配置要模型+harness 一起选**：同一个模型换 harness 能差 14 个点，做选型实验时别固定 harness 只换模型。
- **省钱的配置未必是分数最高的**：Kimi-K3 用少 38% 的工具调用拿到几乎相同的成功率，生产环境值得算这笔账。

最后留一个论文没回答、但我觉得更本质的问题：如果 agent 写 checker 的上限是 45%，那"agent 用 checker"（拿现成分析器扫漏洞、判读告警、定位根因）会不会是更现实的落地路径？扫描、判读、修复的闭环，可能比"从零造分析器"更早到来。

觉得有启发的话，欢迎点赞、在看、转发。跟进最新AI前沿，关注我
