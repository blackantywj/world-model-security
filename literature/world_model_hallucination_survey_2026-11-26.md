# 世界模型幻觉：文献地图、技术路线与具身安全选题

**检索截止：2026 年 11 月 26 日（北京时间）。** 这是定向叙述综述，不是系统综述。主看 2025–2026 年的视觉生成式、动作条件及潜空间世界模型，保留较早的 model-based RL 先例。不把 LLM 事实幻觉、VLM 感知问答、普通 VLA 控制失败当成同一问题。例：HalluWorld [34] 借有真值的参考世界测 LLM 的声明；本报告测的是预测世界动力学的模型本身。通过术语直搜、物理一致性评测、动作干预与低覆盖、推理纠错与闭环安全等入口查找，核对原论文、官方会议页面或作者项目；未确认正式发表的工作只标 arXiv，不以某个榜单推断全部模型的表现。

## 摘要与问题

一个视频可以完全像真的，却给定了错误动作后的未来；一个潜空间模型甚至可以不输出视频，就让规划器把危险动作排在安全动作之前。文献从「视频中发生的事是否符合常识」走向「给定当前状态和动作，预测是否忠实」，并进一步走向「不忠实的预测是否改变决策」。本笔记回答：（1）哪些现象应算幻觉，如何定位？（2）干预训练、状态表示、推理或下游使用，分别能解决什么？（3）接触丰富操作中，什么问题足够窄、可测且能与已有工作拉开差距？

## 1. 操作性定义与诊断

本文把**世界模型幻觉**操作性定义为：给定观测历史与动作，模型对决策相关状态、转移或事件作出具体预测，但预测与可获得的物理、传感器或可复位参考证据矛盾；对随机系统，还需考虑真实的条件结果分布。这个定义用于实验操作，不假定所有论文共用一套术语。概率预测的单次落空不算幻觉；看不见触点或摩擦、存在多个物理可行未来，本身也不是幻觉；可检验的问题是模型是否持续漏掉某一已证实分支、把错误未来赋予过高置信度，或在改变有因果作用的动作后仍强行输出同一个结果。画面噪声只有在扭曲了可检验的状态或事件时才进入本定义。

| 失真位置 | 例子 | 最小诊断与必要对照 |
|---|---|---|
| 观测/状态表征 | 小目标进出遮挡后丢失身份，或视觉 latent 区分不了已经贴合与仍有间隙 | 对真实帧做编解码或状态探针；与真实物体跟踪、接触传感器对齐。探针失效不能直接归咎于 latent，换独立/充分训练的读出对照 |
| 动作→机器人运动 | 改变末端方向、幅度或时序，预测运动仍接近示范 | 固定前缀，替换/置零动作并读取关节或末端轨迹；要求方向与幅度正确，不把「只要有变化」当成忠实 |
| 机器人运动→环境响应 | 末端运动正确但没碰到就推走物体；真实会滑移的动作却预测为稳固 | 匹配实际达到的动作轨迹，单独测接触时刻、物体运动和事件频率；纳入未接触负对照，注意机械臂未跟踪命令造成的假阳性 |
| 多步状态漂移 | 一帧正确，但遮挡后的物体渐渐偏位、消失或重生 | 比较真实历史喂入与自由滚动，按预测时距作图；用新观测重锚后再测，分离一步转移错误和自回归累积 |

Hansen & Wang 的三阶段诊断与上表互补：他们把感知重建、忽略动作、滚动发散分开；上表把第二类再拆成「动作有没有被执行」与「已执行动作有没有产生正确后果」，这正是 WorldSimProbe 的区分。**危险动作被模型判为安全**是上述失真在决策层的表现，不是独立的物理机制；还需排除风险读出器或排序器自身有偏。物体幻觉与安全幻觉也可能不重叠：画错背景但不改变可执行计划，和视频很精美却漏掉滑移，不应同权计分。[1, 2, 5, 18]

## 2. 诊断与基准：看见错误、证实错误、衡量决策后果

| 层级 | 一手代表论文 | 可用来回答 | 不能替代 |
|---|---|---|---|
| 视频物理常识 | VideoPhy / VideoPhy-2 [6, 7]；WorldModelBench [8]；Physics-IQ / Verified [35, 9] | 违反了什么先验，是否遵循指令，视频续写是否和实拍物理一致；Verified 显示修订参考、提示和聚合可改变模型排名 | 给定机器人实际控制量时的动力学真值；不把 VLM 评分等同于传感器证据 |
| 物理泛化与初始状态干预 | PhyWorld [10]；CRONOS [11]；GeoFlip [12]；What-If World [13] | 受控合成动力学能否分布外泛化；不相关外观改变不应改变物理事件；几何或动作参数改变时，预测是否应随之改变 | CRONOS/GeoFlip 的主要协议不是机器人底层动作回放；What-If World 更改文本条件，不等价于执行关节动作 |
| 机器人视频失败定位 | EWMBench [14]；WFM-Eval [15] | 前者拆场景、运动、语义；后者把物体幻觉、时序不一致等失败单独计量，多数据集结果说明应警惕单榜排名 | 由一个因子的独立干预所建立的因果证据 |
| 动作条件模拟器真实性 | ACWM-Phys [16]；WorldSimProbe [2]；MMBench2 [1] | ACWM-Phys 用可控模拟器检查物体与交互动力学泛化；WorldSimProbe 测动作实现与环境响应；MMBench2 用有动作、奖励、live simulator 的数据研究训练覆盖和滚动 | 模拟器参考结果不能替代未见真实物体、传感噪声或接触参数的验证 |
| 规划与闭环效用 | CURE [5]；ContactGuard [17]；When World Models Lie [18]；Risk-Triggered Search [19]；SafeStage / OopsieVerse [44, 45] | 候选动作 regret、接触前中止、预测失真时恢复、以按需搜索改善过程安全 | SafeStage / OopsieVerse 测的是实际执行后果，不判断预测出错在哪一环；仿真约束和少量实机验证不构成安全保证 |

**直接研究幻觉的阅读主线。** [1] 提供辨别 tokenizer、动作、滚动三阶段的实验以及探测、数据重加权和定向采集，对分布覆盖机制最直接。作者与 NeurIPS 官方名单均列为 NeurIPS 2026；论文中少量适配数据指**每个目标任务** 50 条轨迹的实验设定，不应作为通用数据预算。[3] 将问题拓展到无像素监督的潜空间检测：在冻结 DINO-WM 上用真实转移训练分数模型，检测、定位，并尝试修正局部 token；仅验证了两个导航环境，主要纠正证据在单步，不是接触安全保障。「无幻觉标签」不等于「无需真实转移」。[4] 与 WoVR [20] 均关注不真实 rollout 对策略训练的偏置：前者软降权，后者使用关键帧起点和模型–策略共演；这和部署时对一个候选动作实时放行是两种评测。FACT [21] 直接展示成功样本偏置：失败动作也可能被生成成成功抓取；失败轨迹用于监督后果而不是模仿动作。与之相邻的 VLAW [36] 用实机回采迭代提高视频世界模型的接触预测质量，再用生成数据改善策略。[5] 则直接度量决策：有数据支持时，重加权和更多采样有时有助于减少 regret；动作覆盖严重不足时，增加采样不能代替补数据。其结论基于论文的覆盖、候选动作与环境设定，并非通用安全认证。

## 3. 按干预位置梳理技术路线

| 技术路线 | 核心做法与最相关工作 | 适用边界 |
|---|---|---|
| **A. 训练数据覆盖和动作监督** | MMBench2 [1] 用覆盖感知采样和定向回采；FACT [21] 纳入实际失败结果，并使后果预测显式依赖已执行动作；CURE [5] 根据决策区域的覆盖决定补数据还是重加权；VLAW [36] 用真实轨迹回采纠偏 | 有效覆盖不只看数据小时数；同一场景里缺少候选动作、失败边界或恢复分支时，更多成功演示不等价于填补空缺。不可见的状态不能指望只增同类数据就恢复 |
| **B. 表征、记忆和物理观测** | WorldMem / Mem-World [22, 23] 用历史证据缓解遮挡和长时状态遗忘（也可对照 3D 空间记忆 [37]）；VT-WM [24] 用触觉提供视觉缺失的接触证据；PHANTOM [25] 联合建模物理 latent 和视频；HiTac-WAM [26] 直接预测接触等后果并用于动作决策 | 记忆要做错配/无内容对照，不能只用有无记忆的增益来证明模型读对了历史 [27]。如果尚未接触或触觉不能区分关键状态，应允许不确定或获取新证据。表征约束不自动保证真实条件动力学 |
| **C. 推理时诊断、纠错和再观测** | MEND [3] 做局部潜状态检测与修正；WMReward [38] 用潜空间预测信号筛选或引导视频生成；WAV [39] 用逆动力学和前向回环验证状态在动作下是否可达；IMPLY [28] 用真实校准推触锚定所预测的物理参数，审计不同动作分支是否同时与证据相符 | 自洽不等于正确；同一种偏置可出现在全部样本。IMPLY 依赖可用物理反演与锚点，不自动扩展到任意接触任务。纠错后必须重新测动作忠实与校准，不只测 latent 距离 |
| **D. 管控错误模型的使用** | MBPO [29] 从真实状态短滚动，WoVR [20] 从关键帧起跑，HaWMPO [4] 在策略后训练中降低对不可信想象的依赖；ContactGuard、When World Models Lie、RTS [17–19] 分别演示预警中止、观测误差驱动恢复、风险触发搜索；Seo 等 [40] 方法在 latent 与认知不确定性的联合空间做安全过滤 | 下游表现好不表示世界模型更准确。实际部署要区分「预测事件风险」和「判断预测是否可信」，同时报告误中止、时延和原任务效用；独立、可验证的安全约束不宜仅靠模型自评 |

**三个不能混淆的判断。** 第一，覆盖不足是多项研究共同找到的机制，却不等于唯一解释；同一条 RGB 历史下存在不同真实接触状态时，增加 RGB 数据可能改善概率估计，但不能辨认某个具体试次隐藏的状态；这正是触觉和主动观测的价值。第二，物理一致性和视觉一致性都是必要线索，只有与相同初态的候选动作差异对应，才是在测模拟器忠实度。第三，检测到高误差时，修正模型、请求证据、退回保守控制对应不同问题，不能用一个通用置信阈值混为一谈。[1–3, 5, 13, 17–19, 24, 28]

## 4. 优先研究方向

### 首选：动作排序翻转型「危险判安全」幻觉

**拟题：同视不同触条件下，动作条件世界模型的风险排序幻觉：诊断、校准与拒判。** 不把「视触觉 + JEPA + 安全过滤」当贡献：VT-WM 已证明接触证据能改善物理表征，HiTac-WAM 已涉及接触预测与候选动作排序，ContactGuard 已验证短期潜空间预测可触发真机中止，Calibrated Predictive Safety 已提出风险校准和过滤，CURE 已研究覆盖与决策 regret，Imperfect World Models are Exploitable 已在有限 MDP 下把偏好反转形式化 [41]。较具体的差异化是：在**同一可复位接触状态和预注册候选集**内，测量危险动作是否被错误放行或排在安全动作之前，定位是证据不足、模型转移失真、还是风险读出错误，并验证定位后是否能降低闭环损失。[5, 17–19, 24, 26, 30]

- **任务**：选夹持后短程轻搬运，先只研究滑移/滑落一个可由独立传感器标记的事件。固定夹爪、相机、外观和观察时刻，操纵可复位的摩擦/接触状态；使用两至三个可安全测试的动作块。对每种状态随机交替执行候选动作并重复试验；仿真从同一快照分支，实机不声称每次可精确复现同一个微观状态，使用匹配初态和重复估计事件频率。若使用触觉，限制在候选动作前已可获得的信号。真机使用低高度、软承托、速度/力限制和独立停机；不为采集标签刻意执行有损动作。
- **对比**：同观测、同候选集合、同任务目标、同推理预算比较：固定规则/力位阈值、直接动作条件风险预测器、视觉世界模型、视触觉世界模型。各模型配正确动作、动作打乱或无动作；增加无关外观变化、未见物体/摩擦组合、遮蔽触觉对照。SEVI 是直接风险和不确定性估计的先例 [31]。Fast-WAM [42] 也是不预设在线生成必要性的反例，但其任务成功证据不能代替风险评测。若在候选集内根本无法稳定复现风险差异，优先修实验，不先改模型。
- **定位**：在仿真里把真值或对齐的真实未来编码后送进**同一个**风险读出，检查动作排序是否恢复；再比较真值事件指标与模型读出。由真实历史喂入到模型滚动、由正确到打乱动作、由有到无接触信息，分别测量误差变化。若读入真实未来仍判错，不能主张都是世界模型幻觉；替换未来同时改变分布时，需在同编码空间做验证。
- **指标**：将「危险中漏放的比例」与「放行中实际危险的比例」分别定义并报告；匹配对内真实事件频率差可区分时，报告风险排序翻转率，近似平局单独处理；概率报 Brier、分组可靠性图，拒判报风险–覆盖率曲线。闭环报无滑落完成率、误停率、有效预警提前量（扣除推理与执行响应时延）和任务时间。以对象/初态和轨迹为统计单位，不把高相关帧当独立样本；已中止且未回放的候选没有原动作结果，不能标成安全。考虑报告同等任务进度/约束下的风险排序，避免将合理的速度–风险权衡编码成「幻觉」。
- **可证伪判据**：预注册同等放行覆盖/完成效率下，世界模型应比直接风险模型更少漏放且更少排序翻转；跨物体、摩擦变化仍要检查校准。如果没有增益，不宣称世界模型是必要组件；仍可留下可复用的诊断基准。如果触觉只是改善所有模型同等程度，结论应是“观测改善”，而不是“世界模型特有的幻觉抑制”。

### 三条备选

1. **把“不应该变”和“应该变”放进同一动作干预测试。** 不相关的纹理/镜头变化只应改变外观，真正的动作、接触与摩擦变化应改变某些可观测物理事件。用两轴配对数据验证事件频率差的方向/幅度，而不是任何输出差异都算服从；对比 CRONOS、GeoFlip、What-If World 和 WorldSimProbe。若进一步做个体级反事实，需明确固定哪些未观测扰动；受控匹配分支并不自动证明从离线数据识别了反事实。近期已有 CLWM 和 Counterfactual World Models，选题宜限定实物接触、可重复事件和条件假设 [2, 11–13, 32, 33]。
2. **证据驱动的再观测与动态预测窗口。** 在感知歧义时比较重拍、低风险触觉探测和直接拒判，在可观测时比较延长推演与增加样本；用响应该风险的时间预算和等成本的风险–完成率曲线做评测。对照 MEND、IMPLY、CURE、RTS；不把普通不确定性阈值、短滚动或拒判称为新的机制。该方向优先测试能否自动区分“缺证据”“缺训练覆盖”“抽样噪声”三种情形 [3, 5, 19, 28, 29]。

3. **区分「潜空间没有物理信息」和「预测时没用上」。** 用真实与生成未来的配对样本分层探测接触、相对速度和物体身份是否可从特征读出，再通过有控制的特征消融或交换测试动作排序是否随之改变。可读 Interpreting Physics in Video World Models [43]；物理量可解码本身不证明它被预测器或规划器使用。这一方向可先在模拟器或已有轨迹上开展。

## 5. 精读顺序

先读 **Hansen & Wang → WorldSimProbe → CURE**：分别回答错误如何形成、怎样以控制量证伪、怎样改变动作选择。然后按做法选 **MEND + IMPLY**（检测与外部锚定）或 **VT-WM + HiTac-WAM + ContactGuard**（具身接触证据与闭环验证）。如果想做基准，先用一批可复位的配对试验验证标签和风险排序是否可复现，再决定是否训练新模型。

## 6. 小结

**如何测：** 单片段评估物理可行性、匹配分支检查动作与后果、闭环测决策损失；三层不能互换。**如何修：** 对覆盖不足补数据，对观测缺失补证据，对滚动漂移重锚，对风险不明限制模型使用。**先做什么：** 在一个可复位的接触操作里先验证风险排序翻转是否可测，再决定该改世界模型、加触觉，还是直接做风险预测。

## 一手参考

1. Hansen & Wang. [Hallucination in World Models is Predictable and Preventable](https://arxiv.org/abs/2606.27326), NeurIPS 2026（[官方名单](https://neurips.cc/Downloads/2026)、[作者记录](https://www.nicklashansen.com/)）；[作者交互版](https://www.nicklashansen.com/mmbench2/)。
2. Co et al. [WorldSimProbe: Diagnosing Simulator Faithfulness in Action-Conditioned World Models for Embodied Manipulation](https://arxiv.org/abs/2608.09298), arXiv 2026。
3. Alrasheed et al. [MEND: Label-Free Detection, Localisation, and Correction of Latent Hallucination in World Models](https://arxiv.org/abs/2609.39182), DICTA 2026（arXiv 注明录用）。
4. Chen et al. [HaWMPO: Hallucination-Aware World Model-based Policy Optimization for Generalist Robot Policy](https://arxiv.org/abs/2609.09941), arXiv 2026。
5. [Plausible Videos Can Lead to Poor Decisions: Diagnosing and CURE-ing World Models for Planning](https://arxiv.org/abs/2611.22561), arXiv，2026-11-17；[全文](https://arxiv.org/html/2611.22561v1)。
6. Bansal et al. [VideoPhy: Evaluating Physical Commonsense for Video Generation](https://proceedings.iclr.cc/paper_files/paper/2025/hash/fce2d8a485746f76aac7b5650db2679d-Abstract-Conference.html), ICLR 2025。
7. [VideoPhy-2: A Challenging Action-Centric Physical Commonsense Evaluation in Video Generation](https://videophy2.github.io/), ICLR 2026（作者项目）。
8. [WorldModelBench: Judging Video Generation Models As World Models](https://proceedings.neurips.cc/paper_files/paper/2025/hash/4ec03ed08a3fcb59e1c815b5598beff1-Abstract-Datasets_and_Benchmarks_Track.html), NeurIPS 2025 Datasets and Benchmarks。
9. Rädsch et al. [Physics-IQ Verified](https://arxiv.org/abs/2606.18943), arXiv 2026。
10. Kang et al. [How Far Is Video Generation from World Model: A Physical Law Perspective](https://proceedings.mlr.press/v267/kang25g.html), ICML 2025。
11. Begiristain et al. [CRONOS: Benchmarking Counterfactual Physical Consistency in Video Models](https://arxiv.org/abs/2605.23699), arXiv 2026。
12. Li et al. [GeoFlip Preview: Benchmarking Geometry-Sensitive Physics in Video Generation](https://arxiv.org/abs/2610.42002), arXiv 2026。
13. Cai et al. [What-If World: A Causal Benchmark for General World Models in Embodied Scenarios](https://arxiv.org/abs/2605.27589), arXiv 2026。
14. [EWMBench: Evaluating Scene, Motion, and Semantic Quality in Embodied World Models](https://arxiv.org/abs/2505.09694), arXiv 2025。
15. [WFM-Eval: An Evaluation Framework for Video World Models in Robotic Manipulation](https://sahilkhose.github.io/wfm-eval-cvpr26/), CVPR 2026 Video World Models / Failure Modes and Evaluation in the Age of Foundation Models Workshops（作者项目页）。
16. [ACWM-Phys: Investigating Generalized Physical Interaction in Action-Conditioned Video World Models](https://arxiv.org/abs/2605.08567), arXiv 2026。
17. Zheng et al. [ContactGuard: Pre-Contact Execution Monitoring with Action-Conditioned Latent World Models](https://arxiv.org/abs/2608.13438), CoRL 2026（[作者出版清单](https://www.weimingzhi.com/publication-list)）。
18. Cao & Bansal. [When World Models Lie: Adaptive Safety Analysis Under Wrong Imaginations](https://arxiv.org/abs/2609.34300), arXiv 2026。
19. Deng et al. [Risk-Triggered Search: Behavior-Preserving Safe Control](https://arxiv.org/abs/2611.24409), arXiv 2026。
20. Jiang et al. [WoVR: World Models as Reliable Simulators for Post-Training VLA Policies with RL](https://arxiv.org/abs/2602.13977), arXiv 2026。
21. [FACT: Failure-Aware Causal Training for World-Action Models](https://arxiv.org/abs/2608.10232), arXiv 2026。
22. [WorldMem: Long-term Consistent World Simulation with Memory](https://papers.nips.cc/paper_files/paper/2025/file/470629a47e2d65ce0606c40055df5d26-Paper-Conference.pdf), NeurIPS 2025。
23. [Mem-World: Memory-Augmented Action-Conditioned World Models for Persistent Robot Manipulation](https://arxiv.org/abs/2606.18960), CoRL 2026（arXiv v3 注）。
24. Higuera et al. [Visuo-Tactile World Models](https://arxiv.org/abs/2602.06001), arXiv 2026。
25. Shen et al. [PHANTOM: Physics-Infused Video Generation via Joint Modeling of Visual and Latent Physical Dynamics](https://openaccess.thecvf.com/content/CVPR2026/html/Shen_PHANTOM_Physics-Infused_Video_Generation_via_Joint_Modeling_of_Visual_and_CVPR_2026_paper.html), CVPR 2026。
26. Xue et al. [HiTac-WAM: A Hierarchical Tactile World Action Model for Contact-Rich Robot Manipulation](https://arxiv.org/abs/2608.19574), arXiv 2026。
27. [Does Video Memory Use What It Retrieves? A Causal Audit of Memory Specificity](https://arxiv.org/abs/2609.12090), arXiv 2026。
28. [IMPLY: Physically Anchored Consistency for World-Model Rollouts](https://arxiv.org/abs/2609.12441), arXiv 2026。
29. Janner et al. [When to Trust Your Model: Model-Based Policy Optimization](https://proceedings.neurips.cc/paper/2019/hash/5faf461eff3099671ad63c6f3f094f7f-Abstract.html), NeurIPS 2019。
30. [Calibrated Predictive Safety for Heterogeneous Robots: An Action-Conditioned JEPA Framework with Model-Based Safety Shields](https://arxiv.org/abs/2608.17496), arXiv 2026。
31. Shithil & Liu. [SEVI: Safe Evidential Real-World Reinforcement Learning](https://arxiv.org/abs/2610.31829), RA-L 2026（arXiv 注明录用）；[作者项目](https://sevi-safe.github.io/)。
32. [Learning Counterfactual World Models for Embodied Reasoning under Partial Observability](https://arxiv.org/abs/2609.05834), NeurIPS 2026 PTA Workshop（arXiv v2 注）。
33. [Counterfactual World Models: JEPAs That Reason About “What If”](https://arxiv.org/abs/2611.24778), arXiv 2026。

34. Liu et al. [HalluWorld: A Controlled Benchmark for Hallucination via Reference World Models](https://arxiv.org/abs/2605.19341), NeurIPS 2026 Evaluations & Datasets；此处是范围对照。
35. Motamed et al. [Do generative video models understand physical principles?](https://arxiv.org/abs/2501.09038), Physics-IQ 原始论文，arXiv 首版 2025。
36. Guo et al. [VLAW: Iterative Co-Improvement of Vision-Language-Action Policy and World Model](https://proceedings.mlr.press/v306/guo26i.html), ICML 2026。
37. Wu et al. [Video World Models with Long-term Spatial Memory](https://arxiv.org/abs/2506.05284), NeurIPS 2025。
38. Yuan et al. [Inference-time Physics Alignment of Video Generative Models with Latent World Models](https://openaccess.thecvf.com/content/CVPR2026/html/Yuan_Inference-time_Physics_Alignment_of_Video_Generative_Models_with_Latent_World_CVPR_2026_paper.html), CVPR 2026。
39. Liu et al. [World Action Verifier: Self-Improving World Models via Forward-Inverse Asymmetry](https://arxiv.org/abs/2604.01985), NeurIPS 2026（见 [官方名单](https://neurips.cc/Downloads/2026)）。
40. Seo et al. [Uncertainty-aware Latent Safety Filters for Avoiding Out-of-Distribution Failures](https://arxiv.org/abs/2505.00779), CoRL 2025。
41. Bhamidipaty et al. [Imperfect World Models are Exploitable](https://arxiv.org/abs/2605.15960), NeurIPS 2026（见 [官方名单](https://neurips.cc/Downloads/2026)）。
42. Yuan et al. [Fast-WAM: Do World Action Models Need Test-time Future Imagination?](https://arxiv.org/abs/2603.16666), NeurIPS 2026（见 [官方名单](https://neurips.cc/Downloads/2026)）。
43. Joseph et al. [Interpreting Physics in Video World Models](https://proceedings.mlr.press/v306/joseph26a.html), ICML 2026。
44. Luo et al. [SafeStage: Evaluating Safety Before, During, and After Vision-Language-Conditioned Robot Manipulation](https://arxiv.org/abs/2609.21223), arXiv 2026。
45. Balaji et al. [OopsieVerse: A Safety Benchmark with Damage-Aware Simulation for Robot Manipulation](https://www.roboticsproceedings.org/rss22/p098.html), RSS 2026。

