# Jev / Kev 决策模型 × Git 记忆存储 · 调研学习材料

> 2026-09-23 ｜ 三轮并行调研 + 关键来源逐条核实
> 背景：为「agentic 记忆维护系统」寻找理论依据、失效模式、可复用实现与学术背书

---

## 0. 三句话总结

1. **「决策模型 + git 记忆存储」这个组合，开源社区没人做过。** 两条线各自都有人在走，但没有交集——这是空白地带。
2. **但两条线都有扎实的先行工作**：决策模型的架构与失效模式已被拆得很清楚（有人用约 1 万次 API 探测黑盒逆向）；「用小模型做记忆管理」已有 ACL 2026 正刊论文。
3. **三条硬约束从材料里浮出来**：候选**顺序**会改变判断结果、候选**数量**会改变判断结果、**局部维护**优于全局重组。这三条直接影响判断层怎么设计。

---

## 1. 最重要的发现：这是空白地带

调研结论（GitHub 实查 + 多轮检索）：

| 类别 | 有没有人做 | 代表 |
| --- | --- | --- |
| 用 Jev/Kev 做**上下文裁剪/过滤** | ✅ 有 | `tamaratran/jev-pruner`（140★，裁 Bash 输出）、`compozy/yoshi`（25★）、`leonaaardob/fast-dev-compaction`（4★） |
| 用 Jev/Kev 做**任务/风险判定** | ✅ 有 | `thruwire/foreman`（522★，判"任务是否完成"）、`leepokai/jev-guard`（27★，工具调用风险 deny/ask/allow） |
| 用 git 做**长期记忆存储** | ✅ 有多个 | DiffMem、okf-agent-memory、ai-memory-reference、gitmemo 等 |
| 记忆**冲突消解 / 状态机 / 过期清理** | ✅ 有先例 | `fuyuxiang/echo-agent`（矛盾检测+遗忘曲线）、`markhuangai/dense-mem`（conflict detection + provenance）、`diqierjia/StrataGate`（recency decay） |
| **用决策模型做记忆判断 + git 存储** | ❌ **没有** | —— |

而且注意第三条的**方式差异**：现有的冲突消解绝大多数用**大模型 prompt 或时间戳规则**，**没有一个用专用小决策模型**。有的项目（`oleksiijko/pmb`）在 hook 路径上挂了个"亚毫秒分类器"，但那是**正则**，不是模型。

→ **你想做的这个组合，学术上有理论、工程上无先例。** 这不是坏消息——意味着没有现成方案可抄，也意味着这条路没人验证过坑在哪。

---

## 2. 理论侧：Jev 到底是怎么工作的

**来源：`Jev's Architecture Unmasked`（archerhume.com）— 已亲自核实，内容详实可信**

作者用约 **10,000 次 API 探测**（含 6,800 条基准记录、延迟测量、token 计数、行为干预实验）对闭源的 Jev 做黑盒逆向。这是目前对这类模型最深的公开分析。

### 2.1 推断出的架构

| 环节 | 机制 |
| --- | --- |
| 主干 | **因果 Transformer**，作者推测用稀疏 MoE（明确标注为推断非实测——"稀疏专家无法从外部观察"） |
| ① 共享状态编码 | `state` 只编码一次，进前缀 KV cache |
| ② 并行问题编码 | 每个问题分支带自己的指令和选项，通过 attention 读共享 state；**问题之间互不 attend** |
| ③ **列表级选项处理** | 一个问题的候选选项作为**有序列表**共同输入，互相影响（**不是各自独立的 logits**） |
| ④ 直接概率读头 | prefill 后结束，**无自回归解码**。取最终隐藏向量 z=Wh+b，softmax 得分布 |
| ⑤ 训练目标 | RLCD（Reinforcement Learning for **Calibrated** Decisions），用正确评分规则优化预测分布 |

**关键证据**（这些实验设计很值得学）：
- **token 计费严格可加** → 证明 state 只编码一次
- **延迟不随 output token 数变化**：200 选项与 2 选项同速返回（255 选项报 2,714 output tokens）
- **可见性干预**：把密文放进"兄弟问题"→ 概率 0.00；移除兄弟 → 0.00；放进 `state` → 0.90–0.92。**证明问题隔离 + 状态共享**

### 2.2 三条对你直接致命的失效模式

这三条是我认为本次调研**最有价值的发现**，因为它们直接约束你怎么调 Kev。

#### 失效一：位置偏差 —— 同样的证据，放的位置不同，结论不同

48 选项参考试验（2 模板 × 2 值 × 6 排列 × 2 重复）：

| 参考卡放在哪 | 正确率 |
| --- | --- |
| 选项**首部** | 12/16（概率散乱在 ~0.50） |
| 选项**中部** | 11/16 |
| 选项**尾部** | **16/16**（概率 ~0.89） |
| 移入 `state` | **48/48**（1.00） |

工单分类探针：**反转选项顺序**，某分类概率从 **0.84–0.89 移到 0.93–0.96**。

> **对你的意义**：你做召回精排时，候选记忆在 `state` 里的**排列顺序会改变排序结果**。如果阈值设在 0.9 附近，仅因顺序颠倒就可能翻转动作。学术上这现象早有研究（arXiv:2308.11483，MCQ 上重排可造成 13%–85% 的性能差）。

#### 失效二：列表级交互（IIA 违背）—— 加一个无关选项，已有选项的结论就被稀释

10 组随机块实验（每组含 4 选基线、5 选+无关项等），50 请求：

| 条件 | customer/unknown 的对数 odds |
| --- | --- |
| 4 选项基线 | **+0.38** |
| 加入一个无关选项（weather） | **+0.11** |

平均变化 **−0.28**，95% 配对 t 区间 **−0.36 ~ −0.19**，10 组**全部下降**。

> **对你的意义**：如果选项各自独立再 softmax，分母会抵消，odds 不该变。实测变了。意味着**候选记忆的条数会改变精排结果**——所以每次精排的候选数应该**固定**，忽多忽少会让分数不可比。

#### 失效三：`confidence` 字段不是正确率

API 返回的 `confidence` 公式是 `c = (p_max − 1/K) / (1 − 1/K)`——**它只是分布集中度的算术摘要，不是学习到的正确性估计**。分布很集中也可以是 confidently wrong。

> **对你的意义**：阈值不能建立在"confidence 高就可信"的假设上，必须在**你自己的数据上实测**每档 confidence 对应的真实准确率。

### 2.3 作者给应用层的 9 条建议（提炼）

1. 公共证据放 `state`，不要试图跨问题传递
2. 问题分支当**批量作业**调度，不要串行成多轮对话
3. 阈值必须显式基于概率，且在 held-out 工作流数据上持续评估
4. **部署前必须做排列测试（permutation test）**
5. **不要假设选项独立**，选项集要当联合决策界面设计
6. 直接消费概率分布，**不要把 `confidence` 字段当成正确性预测**
7. 跨问题依赖必须在应用层拆多阶段
8. **不要假设请求间确定性**（重复请求有微小差异）
9. JSON 格式化放应用代码，模型只供概率

---

## 3. 学术侧：小模型做记忆管理已被验证（三篇核心论文）

### 3.1 LightMem —— 和你的方案几乎同构 ⭐️

**《Lightweight LLM Agent Memory with Small Language Models》**
arXiv:2604.07798 ｜ **ACL 2026 main** ｜ v3 2026-04-22 ｜ 已核实

- 用 **SLM（小语言模型）** 驱动记忆系统，把 memory retrieval / writing / long-term consolidation **模块化**
- **在线与离线分离**：在线处理受固定预算约束，离线做整合
- 三层记忆：**STM**（短期·即时对话上下文）/ **MTM**（中期·可复用交互摘要）/ **LTM**（长期·整合后的知识）
- **在线两阶段检索：向量粗召回 → 语义一致性精排**
- 离线：抽象可复用交互证据，**增量**整合进 LTM
- 结果：LoCoMo 上比 A-MEM 平均 **F1 +2.5**；**检索中位延迟 83ms**，端到端 581ms

> **为什么这条最重要**：它的"**向量粗召回 → 精排**"两阶段，和你设计的"**FTS/关键词粗召回 → Kev 精排**"是**同一个架构**。而且它证明了：用小模型做这件事，**F1 有提升、延迟只有 83ms**。这是你能拿到的最强学术背书。
>
> 另外它的**在线/离线分离**也值得照搬：写入整合（慢无所谓）走离线，查询精排（要快）走在线。

### 3.2 DuoMem —— 证明微调 4B 模型做记忆是可行的 ⭐️

**《DuoMem: Towards Capable On-Device Memory Agents via Dual-Space Distillation》**
arXiv:2606.29961 ｜ 2026-06-29 ｜ 已核实

- **双空间蒸馏**：
  - **context-space**：用 teacher 生成的高质量程序性记忆替换 student 自己生成的
  - **parameter-space**：在 teacher 的成功轨迹上微调轻量 **LoRA adapter**
- 结果：ALFWorld 上把 **4B 模型从 4.3% → 77.9%**（72B teacher 是 87.1%），**新增可训练参数 <10M**
- 4B 模型比 72B teacher **快 3 倍以上**（wall-clock），适合实时端侧部署
- 跨 8 个模型（2B–72B）消融显示两个蒸馏轴**互补**

> **对你的意义**：这条直接支持你"微调 Kev-4B 适配记忆场景"的想法。**4B + LoRA（<10M 参数）** 这个组合被证明能把端侧记忆能力从废柴拉到接近大模型水平。而且它给了你一个思路：**不只微调参数，还要把"高质量记忆"本身当作 context 侧的训练信号**。

### 3.3 Agent-Native Memory 评测 —— 局部维护优于全局重组 ⭐️

**《Are We Ready For An Agent-Native Memory System?》**
arXiv:2606.24775 ｜ 2026-06-23 ｜ 清华 + 记忆张量团队 ｜ 已核实

- 提出**四模块分析框架**：`memory representation & storage` / `extraction` / `retrieval & routing` / `maintenance`
- 评测 **12 个记忆系统 + 2 个基线**，5 个 workload、11 个数据集
- 核心结论：
  - **没有单一架构在所有场景占优**——效果取决于记忆结构与工作负载瓶颈是否对齐
  - **localized maintenance（局部维护）比 global reorganization（全局重组）成本效益更高**
  - 细粒度消融量化了各模块对 representation fidelity、retrieval precision、update correctness、long-horizon stability 的影响
- 代码开源：`github.com/OpenDataBox/MemoryData`；论文列表：`github.com/OpenDataBox/awesome-agent-memory`

> **对你的意义**：两条。
> ① **"局部维护优于全局重组"** 直接印证了"巡检要增量、局部，不要定期全库重排重写"的设计。
> ② **它的四模块框架可以直接拿来组织你的方案文档**——比我现在用的"捕获/判断/存储/召回"更贴学术口径。
> ③ `awesome-agent-memory` 这个清单值得收藏，是持续跟踪的入口。

---

## 4. 相关论文清单

### 4.1 精排 / 架构先祖（和你的精排层直接相关）

| 论文 | 编号 | 为什么相关 |
| --- | --- | --- |
| **FIRST**: Faster Improved Listwise Reranking with Single Token Decoding | 2406.15657 | **精排的方法来源**——常规 listwise 重排要生成有序标识符序列，FIRST 直接读**第一个生成 token 的 logits** 得排序，用 learning-to-rank 损失优先排准高相关项，**推理加速 50%** |
| **Pointer Networks** | 1506.03134 | Kev 的 pointer head（对每个选项隐藏态打分再 softmax）在机制上的**直系先祖**——输出空间随输入长度变化 |
| **Hydragen** | 2402.05099 | 共享前缀与独立后缀分开算 attention，长共享上下文吞吐提升最高 **32×**——Kev "state 算一次、问题分支复用"的工程基础 |
| **DeFT**: Flash Tree-Attention | 2404.00242 | 树状/多分支解码的共享前缀 KV 融合，同属"一次 prefill、多分支并行读出"谱系 |

> 这三篇（FIRST/Pointer Networks/Hydragen）是 `jaredpalmer/kev` README 自己列的 related work——**顺着它们读，能理解 Kev 为什么长这样**。

### 4.2 校准性研究

| 论文 | 编号 | 要点 |
| --- | --- | --- |
| Revisiting Uncertainty Estimation and Calibration of LLMs | 2505.23854 | 评测 80 个模型，语言自述置信(LVU)校准最好，但**高准确率 ≠ 可靠的不确定性** |
| Calibrating LLMs with Information-Theoretic Evidential Deep Learning | 2502.06351 | 单次前向内给出校准不确定性，缓解小数据过拟合导致的**过度自信** |
| **Generalized Correctness Models** | 2509.24988 | **关键**：LLM 判断自己答案对不对并不比无关模型强，**关键信号来自目标模型的历史预测模式** → **校准要基于你自己 Kev 在该领域的历史表现，而非模型的自我认知** |

### 4.3 失效模式的学术对应

| 现象 | 论文/研究 | 数据 |
| --- | --- | --- |
| 选项顺序敏感 | arXiv:2308.11483（NAACL 2024 Findings） | MCQ 重排后性能差 **13%–85%**，源于位置偏差，且可被用于对抗攻击 |
| 长上下文衰减（位置型） | **Lost in the Middle**（Liu et al. 2023）、Found in the Middle（ACL 2024） | U 形曲线，中间掉 20–30 点 |
| 长上下文衰减（长度型） | **Context Rot**（2026） | 证据固定但输入变长也掉：几百→三千 token，准确率 **0.92 → 0.68** |
| 对抗/误导输入 | arXiv:2511.05919（Injecting Falsehoods）、**PGEA**（EMNLP 2025 Findings） | prompt 注入污染事实召回；verbalized confidence 本身可被操纵 |

> **Context Rot 那条精确对应 Kev 的 384-token 训练上限**——不是巧合，是同一类现象。你的记忆 `state` 应控制在有效长度内，**关键证据放首尾**（和失效一的位置实验一致）。

---

## 5. 存储侧：git 记忆现状

### 5.1 已知项目状态核实（2026-09-23 实查）

| 仓库 | star | 最近推送 | 状态 | 机制变化 |
| --- | --- | --- | --- | --- |
| `opengap`（open-gitagent） | 2955 | 2026-07-02 | 活跃 | 未变 |
| `thedotmack/claude-mem` | 94517 | 2026-09-22 | 活跃 | 未变（SQLite+ChromaDB，**非 git**） |
| `Growth-Kinetics/DiffMem` | 903 | 2026-08-28 | 活跃 | 未变（`git diff` 增量） |
| `open-gitagent/gitagent` | 702 | 2026-08-20 | 活跃 | 未变 |
| `zhangfengcdt/memoir` | 611 | 2026-09-08 | 活跃 | 未变（ProllyTree 模拟 git，默认 File backend） |
| `severity1/claude-code-auto-memory` | 158 | 2026-04-18 | 停滞 | 未变 |
| `keshrath/agent-knowledge` | 18 | 2026-04-21 | 停滞 | 未变 |
| `xChuCx/agent-memory` | 11 | 2026-09-22 | 活跃 | 未变（Go MCP + stage→review→apply） |
| `kingcharleslzy-ai/agent-soul` | 7 | 2026-03-16 | 停滞 | 未变 |
| `fonlan/gitmemo` | 0 | 2026-06-11 | 停滞 | 未变（Skill + `.mem` git 仓库） |
| `petecog/claude-code-memory` | 0 | 2025-06-07 | 已弃 | —— |

**注意**：`fonlan/gitmemo` 已停滞近 4 个月（0 star，说明几乎没人用）。它作为"最小参考实现"仍然有价值，但**不适合直接当底座**——你需要的写入判断、状态机、精排它都没有。

### 5.2 新发现（2026 年 9 月，按相关度排序）

| 仓库 | star | 创建 | 核心机制 | 可借鉴 |
| --- | --- | --- | --- | --- |
| `okf-memory/okf-agent-memory` | 719 | 2026-09-05 | **Git-native** 持久记忆，Go，sub-300µs 内存 BM25 + 嵌入式 MCP + progressive disclosure | ⭐️ 最接近你的方案，**git 作权威存储 + 无外部 DB 的本地检索** |
| `raghuvinta/ai-memory-reference` | 0 | 2026-09-07 | **Markdown + Git 双平面、Git 作权威、分层检索、authority-gated writes** | ⭐️ 架构与你的设计几乎同构（虽然 0 star） |
| `tigerless-labs/agent-memory` | 965 | 2026-09-01 | 纯 Markdown 事实源 + 本地排序检索 + **独立的 "sleep-time Manage 层"** | ⭐️ **把巡检/整合拆成独立离线层**——正是你的"定期巡检" |
| `fuyuxiang/echo-agent` | 1051 | 2026-03 | 四层认知记忆 + **矛盾检测 + 遗忘曲线** + 高风险审批 | ⭐️ 直接对应你的冲突/过期清理 |
| `diqierjia/StrataGate-AgentMemory` | 96 | 2026-08 | **recency decay**：近期详细、旧记忆渐精简、重要记忆常驻 | 记忆衰减的具体实现 |
| `markhuangai/dense-mem` | 39 | 2026-04 | MCP + typed claims + **conflict detection** + evidence provenance | 冲突检测的数据结构设计 |
| `oleksiijko/pmb` | 281 | 2026-05 | SQLite 本地优先 + 去重 + importance decay + hook 路径 sub-ms 分类器（**正则，非模型**） | 对比参照：它证明了 hook 上做判断的可行性 |

### 5.3 关键问题的直接回答

**Q：有没有项目做过"写入时用模型判断取代还是冲突"？**
✅ 有，而且 **`supersede-not-delete + 来源/状态字段` 已是主流范式**。落地实现：`echo-agent`（矛盾检测 + 遗忘曲线）、`dense-mem`（conflict detection + provenance）、`StrataGate`（recency 衰减）、`pmb`（importance decay + 去重）。
**但**：多数用**大模型 prompt** 或**时间戳规则**判定，**没有一个用专用小决策模型**。Elastic 的 agent-memory 博客给出的方案是"LLM 判 contradiction → 标记 `superseded_by`"。
→ **你的"用 Kev 做这个判断"是新做法**（更便宜、更快、可离线），但要注意：**这条路上没有别人的踩坑经验可借**。

**Q：有没有把本地小模型用在记忆判断上？**
- **学术**：很活跃。LightMem（ACL 2026）、DuoMem（4B 端侧蒸馏）、AutoMem（arXiv:2607.01224，记忆管理作为可学习技能）、Forget to Improve（arXiv:2606.25115，用 net-value-per-byte 预算评分做端侧 keep/share/trust）
- **工程**：**基本没有**。生产级仓库多为 LLM prompt 或规则
→ **这正是你的差异化切入点**：学术已验证可行，工程尚属空白。

---

## 6. 生态侧：Jev/Kev 可直接复用的东西

### 6.1 一个很有用的开发路径 ⭐️

`typesafe-ai/system-one-adapter-python`（**271★**，MIT）——**用任意 LLM API 后端冒充 TypeSafeClient 的 drop-in 适配**。

> **意义**：你可以**先用大模型把整条流水线（写入判断 → 状态机 → 精排）打通并验证 schema 设计**，之后再无缝换成本地 Kev。这解决了"Kev 还没装好、但我想先验证设计对不对"的问题。

### 6.2 官方与网关生态

| 入口 | 状态 |
| --- | --- |
| `typesafe-ai/typesafe-sdk-python` | ✅ 208★，MIT，官方 SDK |
| `typesafe-ai/skills` | ✅ 1939★，官方 System One agent skills |
| Vercel AI Gateway（`typesafe-ai/jev`） | ✅ 已确认 |
| Cloudflare Workers AI（`typesafe/jev`） | ✅ 已确认 |
| LiteLLM | ✅ 已确认（价格表含 `typesafe/jev-1.13.0`） |
| **OpenRouter** | ❌ **确认未上架**（实时 API 复核 454 个模型，未返回 typesafe/jev）——印证了之前的判断 |

### 6.3 Kev 微调生态

`jaredpalmer/kev` 仓内 `skills/kev-finetune`（Apache-2.0，v1.1）——在 Modal 上全自动微调 Kev-4B，**H100 约 $1–3/次**。脚本构成：

| 脚本 | 干什么 | 对你的价值 |
| --- | --- | --- |
| `extract_workload.py` | **扫代码里的 Jev/TypeSafe 调用** | ⭐️ 可以从你自己的代码库自动提取"实际在问哪些问题" |
| `convert_data.py` | CSV/JSONL → 训练记录 | 你的标注数据入口 |
| `generate_data.py` | 用 LLM 合成数据 | 冷启动补数据 |
| `plan_size.py` / `split_data.py` | 规模规划 / 数据划分 | —— |
| `kev_modal.py` | 训练 / 校准 / 对比 / 部署 / 清理 | 全流程编排 |

**实测案例**：工单场景 3 个问题、1050 条数据、**15 分钟**训练，Kev-4B 准确率 **67.7% → 73.6%**。

### 6.4 Jev 系应用的架构参考

| 项目 | star | 做什么 | 可借鉴 |
| --- | --- | --- | --- |
| `thruwire/foreman` | 522 | 判"任务是否完成/测试是否充分" | **判定即门控**的架构 |
| `tamaratra/jev-pruner` | 140 | 回传给 LLM 前裁剪冗长输出 | keep/drop 判定范式 |
| `leepokai/jev-guard` | 27 | 工具调用风险打分（deny/ask/allow） | 三档决策 + prompt injection 检测 |
| `shiftynick/jev-axi` | 19 | CLI 工具：pick/rate/check/rank/triage/guard | 六种题型的 CLI 化 |

> 这些虽然都跑在托管 Jev 上，但**因为 API 兼容，Kev 可以无缝替换**。

### 6.5 第三个后端：Laya（2026-09-18 开源，热度远超 Kev）

**`NandhaKishorM/laya`（Convai Innovations）—— 21,369★ / 1819 forks / Apache-2.0 / 发布 6 天即达此规模**（对比 Kev 同期 4,918★）。

| | **Laya** | **Kev** |
| --- | --- | --- |
| 底模 | **ModernBERT 双向编码器** | Qwen3.5 **decoder** + LoRA |
| 参数 | **421M**（英文/微调版）/ 322M（多语言版） | 0.8B / 4B / 9B |
| 机制 | **每个选项一个 `[MASK]` token**，取该位置 hidden state 打分后 softmax | 对选项 `</opt>` 隐态打分 |
| 训练 | **纯策略梯度 RLCD**（REINFORCE + GRPO 风格组基线），无监督交叉熵；奖励为严格适当评分规则 | LoRA + cross-entropy |
| 上下文 | 512（英文）/ 1024（多语言），encoder 支持 8192 | 训练 384/1024，服务 8192 |
| 延迟 | **38.4ms**（P50，25k 问题） | 721ms（M5，新 state） |
| 内存 | **约 1GB**，CPU/Mac 可跑 | 4B 需 32GB Mac 或 GPU |

**HF 三个 checkpoint**（均 Apache-2.0）：`convaiinnovations/laya`（421M，英文，3201♥）、`laya-multilingual`（322M mmBERT，100+ 语言，229♥）、`laya-typed-decisions`（421M 微调版，95♥）。

> 网上「421M vs 322M」的矛盾由此解释：**三个 checkpoint 参数不同**，不是口径打架。

#### ⚠️ 三条必须知道的限制

**① 零样本不可用（官方自己写在 README 里）**

基础 checkpoint 在 typed-decisions 基准上零样本 **0.362**（多语言 0.352），随机基线 0.318，**多数类基线 0.461**。官方原话：

> "**Laya is a fast base to specialise, not a zero-shot decision engine.**"

那个到处被引用的 0.766 是 `laya-typed-decisions` 在该 benchmark **自己的训练划分上微调后**的结果，需要先在 2×T4 上跑 4–5 小时微调 + 拟合温度才能拿到。

**但要正确理解这一点**：速度（38ms）和准确率（0.362）**是两个独立维度**。38ms 由架构决定（双向编码器 + 单次前向，无 decode），**微调只改权重不改架构**——所以微调后依然是 38ms。而"必须微调"这件事对 Kev 同样成立。

**② `confidence` 字段在三种原语之间语义不一致** ⭐️ 这条最要命

| 原语 | confidence 的定义 |
| --- | --- |
| `choice` / `score` | **归一化熵** `1 − H(p)/log(k)`（分布集中度） |
| `noul` | `max(p, 1−p)` |

后果：同样"一边概率 0.9 的二选一"，`noul` 报 **0.90**，`choice` 按熵值只报 **0.53**。而官方文档还建议"≥0.85 自动处理"——**照这个用，同一个判断会得出两个相反的放行结论**。

> **这印证了 Jev 的同类发现**："confidence 只是分布集中度的算术摘要，不是正确率"是**决策模型这个品类的通病**，不是某个实现的疏忽。任何后端接入前，第一步都是把 confidence 语义校准到统一口径。

**③ 高基数分类弱 + `score` 是最弱的原语**

- 选项超过 20 个时，Banking77 仅 **0.425**（Jev 0.870）→ 对 gitev：**bucket 6–7 类、状态 7 类不受影响**；但**若要做"从整个 taxonomy 里选路径"（几十上百个）会直接踩坑**。
- `score` 原语最弱（SST-5 上 **0.372**）→ **而我们的召回精排正好要用 score**。这条必须实测。

#### 其他实测/官方数据

| 项 | 数值 |
| --- | --- |
| 微调版任务内宏平均 | **83.8%**（23,024 问题，ECE 0.060） |
| 微调版零样本任务族 | 65.1%（2,400 问题） |
| 意图识别与路由 | 99.1%（ECE 0.009） |
| 中文（MASSIVE intent） | zh-CN **0.620**（英文 ckpt）/ **0.630**（多语言 ckpt）；zh-TW 0.460 / 0.540 |
| **选择性自动化** | 接受全部 83.8% → 前 80% 置信 **89.4%** → 前 50% 置信 **92.2%** |
| 出厂校准 | 两个 checkpoint 都过度自信；拟合温度后 ECE 0.466→0.081、0.314→0.106；**多语言版未附带拟合好的温度** |
| 语言路由 | **正确性机制而非便利功能**——英文 checkpoint 在高棉语上准确率 **0.000 却报 0.952 置信度**，置信度无法预警语言不匹配，故路由必须在推理前做 |
| 迭代速度 | PyPI 两天发 14 个版本（0.1.0→0.3.4，09-18~09-20），接口仍在动 |

#### ⭐️ 可直接借鉴的设计：act/escalate 决策头

Laya 除了选项打分，还有一个"该行动还是该升级人工"的决策头，**成本矩阵**如下：

| 情况 | 奖励 |
| --- | --- |
| 自动执行且正确 | **+1.0** |
| 自动执行但错误 | **−3.0** |
| 升级给人工 | −0.5 |

策略**自动学会"置信度 ≥62.5% 才行动"**——阈值不是拍的，是从成本矩阵推出来的。

> **对我们的直接价值**：这比"拍一个阈值"更好，**可以直接替换我们 Auto-Rate@ε 里那个手工选的 τ**。而且它的成本比（**错误代价 = 3 倍人工代价**）与我们「宁可留脏不可错删」的第一原则同源。
>
> 它的**选择性自动化覆盖率-准确率表**（丢掉一半低置信判断，准确率 83.8% → 92.2%）正是 Auto-Rate@ε 的实证数字，已引入 [evaluation.md](evaluation.md)。

#### 归属争议（两篇被引论文均已亲自核实）

作者声称自己先提出该架构，引了两篇论文。实际内容：

| 论文 | 实际标题 | 与「通用非自回归决策架构」的关系 |
| --- | --- | --- |
| arXiv:2503.23303（2025-03-30） | **SalesRLAgent: A Reinforcement Learning Approach for Real-Time Sales Conversion Prediction and Optimization** | ❌ 是**销售转化预测应用**，不是通用架构 |
| arXiv:2510.01237（2025-09-23） | **Confidence-Aware Routing for LLM Reliability Enhancement: ... Pre-Generation Hallucination Mitigation** | ❌ 讲**生成前置信路由**，主题无关 |

严格说，作者原话是"源自 2025 年的研究，**重构原有方案后扩展为**通用 System 1 模型"——所以这是**原创性主张的范围问题，不是抄袭**。但结论明确：**选型理由里不要写"原创性/学术优先权"**（公开记录不支持）；能确凿说的是**代码与权重可下载**。

---

## 7. 六条设计教训（我认为最该记住的）

| # | 教训 | 来源 | 对方案的直接动作 |
| --- | --- | --- | --- |
| 1 | **候选顺序会改变排序结果** | Archer Hume 参考卡实验（首/中/尾 12:11:16）、arXiv:2308.11483 | 精排必须做**位置校准**：多次随机化求平均，或用 `/v1/systemone/permute` 先验证敏感度 |
| 2 | **候选数量会改变判断**（IIA 违背） | Archer Hume（−0.28 log-odds，10/10 组都降） | **固定每次精排的候选条数**，否则分数不可比 |
| 3 | **`confidence` 不是正确率** | Archer Hume（公式只是分布集中度） | 阈值必须**在自己数据上实测**，不能信自报 |
| 4 | **校准要基于自己模型的历史表现** | arXiv:2509.24988（Generalized Correctness Models） | 微调 + 本地校准不是可选项，是**必需项** |
| 5 | **局部维护优于全局重组** | arXiv:2606.24775（12 系统评测） | 巡检要**增量、局部**，不要定期全库重排 |
| 6 | **在线/离线要分离** | LightMem（ACL 2026） | 写入整合走**离线批处理**（慢无所谓），查询精排走**在线**（要快，83ms 是标杆） |

---

## 8. 对现有方案的修正建议

基于以上材料，我建议对之前的设计做这几处调整：

1. **精排层加"位置校准"** —— 不能假设候选顺序无关。先跑 `/permute` 测敏感度，敏感就做多轮随机化取平均（代价是 ×N 倍延迟，要注意）。
2. **固定精排候选数** —— 统一每批的候选条数（比如恒定 8 条），避免 IIA 效应让分数不可比。
3. **巡检改成"局部 + 增量"** —— 按 arXiv:2606.24775 的结论，别做全局重组。参考 `tigerless-labs/agent-memory` 的 "sleep-time Manage 层"，把巡检做成独立离线层。
4. **在线/离线分离** —— 写入整合（含冲突消解）离线跑，查询精排在线跑，目标是向 LightMem 的 83ms 看齐。
5. **阈值必须实测校准** —— 分三档：高置信自动、中档人工 review、低置信不动。实测每档的真实准确率再加入。
6. **开发路径换成两段式** —— 先用 `system-one-adapter-python` + LLM 把 schema 和流水线验证通，再换本地 Kev-4B。这样不会卡在环境搭建上。
7. **微调数据从 `extract_workload.py` 起步** —— 让脚本扫你自己的代码/历史，提取"实际在问哪些问题"，而不是凭空设计。

**还有两个可以白拿的资产**：
- `github.com/OpenDataBox/awesome-agent-memory` —— 持续跟踪这个领域的入口清单
- `github.com/OpenDataBox/MemoryData` —— 12 个记忆系统的评测代码，可以用它当基准框架

---

## 9. 核实状态说明

| 来源 | 核实方式 |
| --- | --- |
| `Jev's Architecture Unmasked`（archerhume.com） | ✅ 亲自抓取，全文提取（8 组实验、上万次探测） |
| arXiv:2606.24775《Are We Ready For An Agent-Native Memory System?》 | ✅ 亲自抓取 abs 页，标题/作者/摘要/提交日期已核 |
| arXiv:2604.07798《Lightweight LLM Agent Memory with SLMs》(LightMem) | ✅ 亲自抓取，确认 ACL 2026 main |
| arXiv:2606.29961《DuoMem》 | ✅ 亲自抓取，数字已核 |
| `jaredpalmer/kev` 仓库与 README | ✅ 亲自用 gh CLI 抓取（4918★，Apache-2.0） |
| 各 GitHub 项目 star / 推送时间 | ⚠️ 由调研子代理用 gh CLI 实查，**未逐一二次核实**，引用前建议复核 |
| arXiv 2406.15657 / 1506.03134 / 2402.05099 / 2404.00242 等 | ⚠️ 由子代理检索，**编号未二次核实**，但均为 Kev README 自己列的 related work，可信度较高 |
| 生态项目清单（jev-pruner / foreman / yoshi 等） | ⚠️ 子代理实查，未二次核实 |

---

## 10. 下一步

按 §8 的七条修正，把设计方案更新为 **v0.2**。主要重写：

- **判断层**：从 Jev 换成 Kev-4B（本地），补上位置校准与固定候选数
- **召回层**：粗召回（`git log --grep` + FTS）→ 精排（Kev，含位置校准）
- **维护层**：状态机 + 局部增量巡检（独立离线层）
- **校准层**：阈值实测 + 微调闭环（`--init_from` + `kev-finetune`）
- **框架**：改用 arXiv:2606.24775 的四模块口径（representation&storage / extraction / retrieval&routing / maintenance）
