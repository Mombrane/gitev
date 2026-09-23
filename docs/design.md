# gitev · 架构设计 v0.2

> 2026-09-23 ｜ 基于 memoir claude-code 插件源码 + Claude Code Auto-Dream 公开设计 + Jev/Kev 能力边界
> 目标：可用的 MVP —— 装一个插件，填 Jev key + git 仓库链接，即可运行

## 第一版范围（v0.1 实现范围）

**只适配 Jev，不接本地 Kev。**

理由：先把端到端链路跑通，避免卡在 Python / uv / 模型下载的本地环境搭建上。判断层做成**可替换接口**，Kev 作为后续版本接入（见 §3 的分工表，接口设计已预留）。

| 能力 | v0.1 | 后续 |
| --- | --- | --- |
| 判断层后端 | ✅ Jev（云端 API） | Kev-4B 本地 |
| hook 抽取 / 注入 | ✅ | —— |
| git 存储 | ✅ | —— |
| 状态机 | ✅ | —— |
| 召回（粗 + 精排） | ✅ | —— |
| 整理层（/dream） | ⏳ 可后置 | —— |
| 微调校准 | ❌ | Kev 阶段再做 |

---

## 0. 一句话结论

**你的四层设想是对的，而且不是凭空的——它和 Claude Code 的记忆架构是对齐的。** 更巧的是，`memoir` 的 claude-code 插件已经把「hook 抽取 + hook 注入 + skill 维护」三层实现得很完整，可以直接当蓝本；「auto dream」有 Claude Code 的公开设计（四道门 + 四阶段）可参考。

我们要做的增量是**两件事**：把**判断层**插进去（v0.1 用 Jev，Kev 留接口），把**状态机**补上。这两件事，现有的开源实现里都没有。

---

## 1. 你的架构 vs 现有实现

| 你的设想 | 有没有现成参照 | 评价 |
| --- | --- | --- |
| hook 级别记忆抽取 | ✅ `memoir` 的 `Stop` hook（async，180s）+ `stop_capture.tmpl` | **直接可抄**，prompt 设计非常成熟 |
| hook 注入 | ✅ `memoir` 的 `SessionStart` hook | **直接可抄**，做了 10 件事 |
| agentic skill 级别召回/读写/维护 | ✅ `memoir` 的 `memory-recall` skill + `remember` 命令 | **直接可抄**，含交互式冲突消解 |
| agentic auto dream 入口 | ✅ Claude Code Auto-Dream 的公开设计 | **可抄设计**，四道门 + 四阶段 |
| 写入/召回用 Jev 或 Kev 做增强 | ❌ 无人做过 | **这是增量** |
| 状态机维护 | ⚠️ gitmemo 有 `status` 字段雏形，memoir 只有 merge policy | **这是增量** |

---

## 2. 整体架构：四层 + 一个贯穿的判断层

```
┌─ 捕获层 ─ Stop hook（async，不阻塞）────────────────┐
│  会话结束时：抽取候选事实 → 判断层 → 写进 .mem      │
├─ 注入层 ─ SessionStart hook ────────────────────────┤
│  会话开始时：注入记忆索引 + 库状态 + 待办提示        │
├─ 维护层 ─ Skill / Slash Command（agentic）──────────┤
│  按需：召回 / 手动写入 / 冲突消解 / 状态流转         │
├─ 整理层 ─ /dream 入口（手动 + 自动门）──────────────┤
│  周期性：巡检 / 整合 / 修剪（垃圾回收器）            │
└─────────────────────────────────────────────────────┘
              ▲          ▲          ▲
        ┌─────┴──────────┴──────────┴─────┐
        │  判断层：Kev（本地）             │
        │  分类 / 冲突判定 / 相关性精排    │
        └──────────────────────────────────┘
```

**为什么判断层要单独抽出来？** 因为四个层都要做判断，但判断的类型不同、成本敏感度不同。抽出来之后，它可以是 Kev（本地免费）、也可以回退到 LLM（兜底）、还可以先不接（MVP 第一步）。

---

## 3. 判断层：Kev 插在六个点

**关键分工原则**：**Kev 不生成文本，只做选择。** 所以"抽取事实"和"整合成文"必须靠 LLM，"分类/判定/排序"交给 Kev。

| # | 判断点 | 题型 | 谁做 | 为什么 |
| --- | --- | --- | --- | --- |
| 1 | 从对话抽取候选事实 | 生成 | **LLM** | 需要生成能力，Kev 做不到 |
| 2 | 门控：这条值不值得记 | noul / 规则 | LLM 主 + **Kev 复核** | memoir 的 prompt 已用规则做得很好了，Kev 用于兜底 |
| 3 | 分类：归到哪个 taxonomy path | choice | **Kev** | 固定分类空间、可微调、零成本 |
| 4 | **冲突判定**：与已有记忆是什么关系 | choice | **Kev** ⭐️ | new / duplicate / supersede / update / contradict |
| 5 | 重要性打分 | score | **Kev** | 用于优先级排序 |
| 6 | 召回精排：候选与查询的相关性 | score + noul | **Kev** ⭐️ | 批量打分，每题独立 |

第 4 和第 6 是 Kev 的主场，也是现有实现完全没有的部分。

### 为什么不让 LLM 全包了？

memoir 的做法是**一次 haiku 调用同时完成抽取 + 分类**（输出 `<path>\t<fact>`），然后绕过底层库的分类器链，端到端快 25–30 倍。这个优化很漂亮，但它有三个代价，而 Kev 正好补上：

| 问题 | LLM 一次性做完 | 用 Kev 做判断层 |
| --- | --- | --- |
| 判断漂移 | 每次调用可能给出不同分类 | 固定权重，输出稳定 |
| 成本 | 每轮都要调 API | 本地零成本 |
| 校准 | confidence 是自报的，不可信 | **可微调 + 可实测校准** |
| 离线 | 依赖 API 可用性 | 完全离线 |
| 冲突判定 | LLM 单轮判"与已有记忆的关系"容易敷衍 | choice 题型 + 候选进 state，判定更可靠 |

**所以建议的终态是**：LLM 只做"抽取"这一件它不可替代的事，**其余全部判断交给 Kev**。这既省钱又稳定。

---

## 4. 状态机设计 ⭐️

这是现有实现里**没有**的部分。设计要点：

### 4.1 状态定义

| 状态 | 含义 | 召回时 | 怎么进入 |
| --- | --- | --- | --- |
| `active` | 当前有效 | ✅ 参与 | 默认 |
| `superseded` | 被新记忆取代（**必须有指向者**） | ❌ 不参与，但可溯源 | 写入时判 `supersede` |
| `patched` | 部分过期，已打补丁 | ✅ 参与（用补丁后的内容） | 写入时判 `update` |
| `contradict` | 与另一条互相矛盾，未判定 | ⚠️ 两条都召回并提示冲突 | 写入时判 `contradict` |
| `pending` | Kev 判不准，等人看 | ❌ 暂不参与 | 置信度落中档 |
| `expired` | 时效性过期（环境变了） | ❌ 不参与 | **巡检**判定 |
| `archived` | 人工归档 | ❌ 不参与 | 人工 |

### 4.2 状态存在哪

**存在 entry 的 YAML frontmatter 里。** 这是 gitmemo 已有的做法（它已有 `status: done` 字段），我们在它上面扩展：

```yaml
---
id: mem_20260923_153045_a3f2
status: active              # ← 状态机字段
created: 2026-09-23T15:30:45Z
last_verified: 2026-09-23T15:30:45Z   # ← 巡检用
verified_by: kev-4b@1.13
confidence: 0.87            # ← Kev 给的置信度
taxonomy: preferences.coding.style
tags: [coding, style]
supersedes: []              # ← 它取代了谁
superseded_by: null         # ← 谁取代了它
related_keys: [...]         # ← 姐妹条目（跨分类）
source: session/2026-09-23#turn-42    # ← 溯源
---

（正文）
```

### 4.3 状态转移表

| 当前状态 | 事件 | Kev 判定 | 目标状态 | 动作 |
| --- | --- | --- | --- | --- |
| （不存在） | 新事实写入 | `new` | `active` | 新建 entry + commit |
| `active` | 写入相似事实 | `duplicate` | `active` | 不新建，只更新 `last_seen` |
| `active` | 写入新事实 | `supersede` | 旧→`superseded` / 新→`active` | 旧条目写 `superseded_by`，新条目写 `supersedes`，**两次 commit** |
| `active` | 写入补充事实 | `update` | `patched` | 旧条目打补丁（保留原值 + 追加变更记录） |
| `active` | 写入矛盾事实 | `contradict` | 双方→`contradict` | 加冲突标记，**进 `pending` 队列等人裁决** |
| `active` | Kev 置信度落中档 | —— | `pending` | 进 staging，等人工 review |
| `active` | 巡检：环境已变 | `expired` | `expired` | 标记 + 保留正文 |
| `contradict` | 人工裁决 | —— | 胜者 `active` / 败者 `superseded` | 记录裁决理由 |
| 任意 | 人工归档 | —— | `archived` | 不再召回 |

### 4.4 三条铁律

1. **永不物理删除。** 只改状态字段，正文留在 git 历史里。`superseded` / `expired` / `archived` 的条目随时可用 `git log` 翻出来恢复——**这是 git 存储相对向量库最大的优势，要用足。**
2. **宁可留脏，不可错删。** 判错的代价不对称：漏掉一条过期记忆只是检索质量略降；错删一条有效记忆是永久丢信息。所以低置信度**什么都不做**。
3. **状态转移必须可溯源。** 每次转移都在 commit message 里写清"由什么触发、Kev 判定什么、置信度多少"。

### 4.5 状态机治不了的那一类

状态机的驱动力是**写入事件**——有新东西来才触发重判。但有一类过期**没有写入事件**：项目路径改了、工具版本升级了、外部依赖废弃了。那条记忆本身没错，只是世界变了，它会安静躺在 `active` 里烂掉。

→ **所以必须有整理层（§7）做定期巡检**，基于 `last_verified` 的时间衰减 + Kev 重新判断"这条现在还成立吗"。

**状态机 + 定期巡检，缺一不可。**

---

## 5. 记忆抽取设计

### 5.1 核心可以照抄 memoir 的 prompt

我读了 `memoir/hooks/prompts/stop_capture.tmpl` 全文，它的设计非常成熟，值得直接借鉴：

**① 「沉默是默认」原则**

> Your default answer is NOTHING. Empty output. Zero lines. Silence.
> 绝大多数轮次没有持久事实，**沉默是正确的、预期的、高质量的结果——它不是失败**。

**② 四道持久性检查**（全部 YES 才输出）

1. 这个事实是否 DURABLE（一周/一月后仍相关）？
2. 未来会话是否真会受益，还是显而易见/短暂的？
3. 是否**不能**从代码、git log、已有文档里发现？
4. 一位资深工程师会写进 onboarding 笔记，还是会翻白眼？

**③ 明确的正向触发器**（覆盖沉默默认）

- 长期规则 / going-forward 指令：`from now on…`、`always X`、`never X`、`every time…`
- 明示偏好：`I prefer…`、`use X over Y`、`don't use…`
- 本轮**已解决**的决策——**抓 why 不只抓 what**；跳过明确推迟的（TBD / pending review）
- 本轮浮现的项目事实
- 非显然的技术知识：不变量、坑、隐藏约束

**④ 明确的排除清单**

- 常规问答、代码阅读、"show me X" 请求
- 当场解决的一次性调试
- **工具调用及其输出**（那是机制不是事实）
- 已有内容的复述、礼貌闲聊、只针对本轮的反馈
- **飞行中的讨论 / 未解决的决策**

**⑤ 一个精妙的边界**（原文特别强调）

> 以**反馈形式**表达的长期规则（"don't do X anymore"、"from now on do Y"）是 DURABLE——**要捕获**。判断标准是规则是否适用于未来轮次，而不是它是否以纠正的形式表达。

**⑥ 输出格式**

```
<taxonomy-path>[<TAB><fact>]
```
- 每行 `路径<TAB>事实`，**0–6 行**
- 路径严格遵守 `^[a-z][a-z0-9_]*(\.[a-z0-9_]+){2}$`（三层、小写下划线、**禁止连字符**）
- **不匹配的行会被下游静默丢弃**——这个正则校验在 `stop.sh` 里也硬编码了一遍，防模型跑偏
- 一个事实可挂多个路径（逗号分隔，最多 2 个），用于真正跨类目的事实
- 无前言、无解释、无"未找到事实"消息——只有行，或完全空

### 5.2 分类体系换成「个人记忆」版

memoir 的 fallback taxonomy（下面是它的原版，是个人向的，很适合你）：

```
profile.{personal,professional}      身份、人口统计、职业、教育、技能、位置
preferences.{coding,tools,work,...}  编辑器、语言、框架、AI 模型、工作方式
workflow.{coding,devops}             测试、分支、评审、部署、版本
context.project.{stack,repo,infra,database,cicd,standards}
relationships.{family,friends,professional}
goals.{career,education,projects,financial}
experience                           过往工作、里程碑、决策
knowledge.technical
behavior.work                        日程、习惯
routine.daily                        站会、仪式
```

**注意它用的是「3 层固定深度」**（`category.subcategory.type`），这带来两个好处：召回时可以按层级渐进展开（§6），以及分类空间固定 → **正适合 Kev 的 choice 题型微调**。

### 5.3 与 Kev 的接法

```
对话转录
   ↓
[LLM 一次调用] 抽取 + 初步分类（照抄 memoir 的 prompt 结构）
   ↓  输出 candidates: [{path, fact, confidence?}]
   ↓
[Kev 批量判定]  ← 这是新增的
   ├─ bucket:    choice  → 六个分类（含 skip）
   ├─ overlap:   choice  → new / duplicate / supersede / update / contradict
   ├─ importance: score  → 1–5
   └─ verify:    noul   → 这条是用户明确确认的事实，还是会话中的推测/转述？
   ↓
按 §4.3 状态转移表执行
```

**注意 Kev 的问题之间是隔离的**（每个问题只看 state、看不到其他问题的答案），所以上面四个问题必须**对同一个 state 独立成立**，不能设计级联逻辑。这正好符合我们的需求。

---

## 6. 召回设计

`memoir` 的 `memory-recall` skill 设计密度极高，直接借鉴：

### 6.1 六种模式 + 默认单发

| 模式 | 何时用 | 动作 |
| --- | --- | --- |
| `[mode=get]` | 查询点名了确切路径 | 直接 get |
| **`[mode=fast]`** | **默认** | 一次 `summarize --depth 3` + 一次批量 `get`，**2 次 CLI 调用 + 2 轮推理** |
| `[mode=drill]` | 仅当 >1000 条 且查询宽泛 | 按 L1→L2 逐层下钻，**每层一次批量调用** |
| `[mode=flat]` | 仅当 >1000 条 且单一 glob | 一次 `summarize --keys` |
| `[mode=blame]` | 溯源问题（"我什么时候决定的"） | git blame |
| `[mode=diff]` | 跨提交/分支问题 | git diff |

**硬上限：一次只返回 5–7 条**，绝不更多。**原始输出不做任何综合**——不分组、不复述、不加"bottom line"、不加 markdown 标题。调用它的父代理负责渲染。

### 6.2 成本启发式（很实用）

> "一次没用上的召回成本很低；漏掉一个记住的偏好成本很高。"
> → **默认开启召回**，只有机械的单符号查找、一次性草稿、用户明确关闭时才跳过。

### 6.3 召回分两阶段，Kev 管第二阶段

| 阶段 | 手段 | 哲学依据 |
| --- | --- | --- |
| 粗召回 | `git log --grep`（commit message）+ 全文索引 | **grep-over-rag**：对结构化、人类可读的文件，按名字/关键词推理优于不透明向量匹配。Claude Code 明确拒绝了向量搜索（它调 `ls()` 然后推理哪些文件相关） |
| 精排 | **Kev**：把查询 + 5–10 条候选塞进 state，`score` 打相关度 + `noul` 过时效 | 语义相关性判断 |

### 6.4 精排的两个工程约束（来自 Jev 失效模式实测）

1. **候选顺序会改变结果** → 用 `/v1/systemone/permute` 先测敏感度；敏感就多轮随机化取平均（代价是延迟 ×N）
2. **候选数量会改变判断**（IIA 违背：加一个无关选项，已有选项 log-odds 从 +0.38 掉到 +0.11）→ **每次精排固定候选条数**

---

## 7. 整理层：Auto Dream（垃圾回收器）⭐️

这是你提的"引导模型去整理仓库级别的记忆"，Claude Code 的 Auto-Dream 有完整公开设计可直接迁移。

### 7.1 四道门（**按计算成本从便宜到贵排序，任一不过立即退出**）

| # | 条件 | 检查成本 |
| --- | --- | --- |
| 1 | 距上次整理 ≥ 24 小时 | 时间戳比较（最便宜） |
| 2 | 累积 ≥ 5 个会话 | 扫文件列表 |
| 3 | 距上次扫描 ≥ 10 分钟 | 时间戳比较 |
| 4 | 获取文件系统锁（防并发） | 需要加锁 |

Claude Code 的实现里，锁文件防止两个实例同时对同一个项目做梦；失败自动回滚。

### 7.2 四阶段

| 阶段 | 做什么 | Kev 能加速的地方 |
| --- | --- | --- |
| **Orient（定向）** | 读索引 + 扫主题文件的**标题和目录**，**不读全文** | —— |
| **Gather（采集）** | **grep 优先，不是读全文**。定向搜：用户纠正过的点、明确保存（"remember this"）、跨会话重复主题、重要决策 | —— |
| **Consolidate（整合）** | 相对时间戳转绝对日期；**删除被明确矛盾的事实**；合并重复观察为规范条目；**解决不同会话的冲突结论** | ⭐️ **两两冲突检测**——这是典型的 choice 判定，且要跑很多对，Kev 便宜 |
| **Prune（修剪）** | 重写主题文件；重建索引（在上限内）；删除不再相关的条目 | ⭐️ 判"这条还成立吗"（noul） |

**实测数据**（Claude Code）：小测试 1 分 19 秒，把 280 行索引减到 142 行；913 个累积会话约 8–9 分钟后台完成。

### 7.3 安全约束（必须照搬）

- **做梦期间对项目代码只读**，只能写记忆文件
- **沙箱在记忆目录内**，不能写到别处
- **失败回滚**
- 顺序化包装，防重叠运行

### 7.4 三层记忆层级

| 层 | 内容 | 加载策略 |
| --- | --- | --- |
| **L1 索引** | 约 150 字符的指针清单，**有行数上限** | 始终在上下文 |
| **L2 主题文件** | 实际知识 | 按需加载 |
| **L3 会话日志** | 原始转录 | **从不全文重读，只 grep 特定标识符** |

> 原设计里有一句话很到位：**"dream 是这个架构的垃圾回收器。没有它，L2 会变陈腐和自相矛盾，L1 会膨胀到没用，L3 会变得无法导航。"**

### 7.5 入口形态

- **手动**：`/dream` 命令 + "consolidate my memory files" 这类自然语言触发
- **自动**：四道门都过时，由 SessionStart hook 提示或后台触发

---

## 8. git 存储结构

```
<记忆仓库>/
├── .git/
├── index.md                 # L1：索引（有行数上限）
├── entries/                 # L2：记忆条目
│   ├── mem_20260923_153045_a3f2.md
│   └── ...
├── topics/                  # L2：主题聚合（dream 维护）
│   ├── preferences-coding.md
│   └── ...
├── staging/                 # 待人工 review（pending / contradict）
├── logs/                    # L3：按日期的 append-only 日志
│   └── 2026/09/2026-09-23.md
└── taxonomy.yaml            # 分类体系（可自定义覆盖）
```

**两种存储模型的选择**：`memoir` 用 key-value（taxonomy path 为 key，反复更新 + merge policy），`gitmemo` 用 append-only 条目文件（每条一个文件）。建议**混合**：

- **一个 taxonomy path ↔ 一个 entry 文件**（兼顾 key 的稳定性与文件的独立可审计）
- 同一 path 的后续变更 = **打补丁 + 状态流转**，而不是覆盖（保留演进历史）

**commit message 规范**（照抄 gitmemo 的思路）：

```
[<taxonomy-l1>] <动作> <对象>

<1-3 句摘要>
status: active → superseded
by: kev-4b@1.13 (confidence 0.91)
trigger: session/2026-09-23#turn-42
```

---

## 9. MVP 配置形态

你要的"填两样东西就能用"，具体是：

| 配置项 | 必填 | v0.1 取值 | 说明 |
| --- | --- | --- | --- |
| **Jev API key** | ✅ | `TYPESAFE_API_KEY` | v0.1 唯一支持的判断层后端；未配置时降级为纯规则模式（仍可捕获，无增强判断） |
| **记忆仓库** | ✅ | git URL 或本地路径 | 空仓库也自动 init |
| **模型版本** | ❌ | 默认 pin `jev-1.13.0` | **必须 pin 版本号而非 `jev-latest`** —— 别名会漂移，会让已校准的阈值失准 |
| **分类体系** | ❌ | 内置默认 / 自定义 yaml | 内置个人记忆 taxonomy |
| **开关** | ❌ | 捕获 / 注入 / 自动整理 | 各自独立开关，互不影响 |

**为 Kev 预留的接口**：判断层后端抽象成一个 `judge` 接口（输入 state + questions，输出 typed answers）。v0.1 的实现是 `JevJudge`（HTTP 调 `api.typesafe.ai`）；后续加 `KevJudge`（调本地端口）时，只需换实现，上层逻辑不变。因为 Kev 的 API 与 TypeSafe System One **完全兼容**，这个替换是 `base_url` 级别的工作。

**借鉴 memoir 的三个设计**：
1. **每条路径独立失败**：capture / 注入 / 整理三条线互不阻塞，各有 escape hatch
2. **幂等初始化**：store 不存在就自动建，不报错
3. **优雅降级**：Jev 不可用（无 key / 超限 / 网络问题）时仍能用规则捕获，只是少了增强判断

**一个已知的部署张力**：Kev 需要 Python 3.12+/3.13 + uv + transformers≥5.17 + 常驻服务。如果是给自己用没问题；如果要给非技术用户装，得把这个藏起来（打包二进制、或默认 Jev 云端 + 可选本地）。

---

## 10. 参考实现对照速查

| 需求 | 去看这个 | 位置 |
| --- | --- | --- |
| 抽取 prompt 怎么写 | `memoir` 的 `hooks/prompts/stop_capture.tmpl` | ⭐️ 直接抄 |
| Stop hook 怎么编排（async + 三路独立 + 绕开内部分类器） | `memoir` 的 `hooks/stop.sh` | ⭐️ 直接抄 |
| SessionStart 注入什么（10 件事 + 防打扰提议） | `memoir` 的 `hooks/session-start.sh` | ⭐️ 直接抄 |
| 召回 skill 怎么写（6 模式 + 5–7 上限 + 原始输出） | `memoir` 的 `skills/memory-recall/SKILL.md` | ⭐️ 直接抄 |
| 冲突消解怎么交互（4 策略） | `memoir` 的 `commands/remember.md` | ⭐️ 直接抄 |
| 分支合并/清理 | `memoir` 的 `commands/sync.md` | 参考（个人记忆可简化） |
| entry frontmatter 格式 | `memoir/gitmemo` 的 `SKILL.md` Entry Format | 扩展 `status` 字段 |
| 文件写入避坑（CJK 编码） | `gitmemo` 的 SKILL.md：**优先用 IDE 原生写文件工具，别用 heredoc** | ⭐️ 重要 |
| 离线整理（四道门 + 四阶段） | Claude Code Auto-Dream 公开设计 | ⭐️ 设计参考 |
| 状态机 + supersede 先例 | `fuyuxiang/echo-agent`（矛盾检测 + 遗忘曲线）、`markhuangai/dense-mem`（conflict detection + provenance） | 参考 |
| 独立离线整理层 | `tigerless-labs/agent-memory`（sleep-time Manage 层） | 参考 |

---

## 11. 实施路线（v0.1，五步，每步可独立验证）

| 步骤 | 做什么 | 验收标准 |
| --- | --- | --- |
| **1. 骨架** | hook 抽取 + git 写入 + SessionStart 注入，**先不接任何模型**（用规则兜底） | 一轮对话后记忆仓库里出现一条格式正确的 entry，新会话能看到它 |
| **2. 接 Jev** | 实现 `JevJudge`：分类（choice）+ 冲突判定（choice）+ 重要性（score） | 写入 20 条测试记忆，分类准确率和冲突判定准确率有数 |
| **3. 状态机** | frontmatter 状态字段 + 转移逻辑 + staging 人工关卡 | 写入一条冲突记忆时，旧条目正确转入 `superseded` 且可溯源 |
| **4. 召回** | 粗召回（`git log --grep` + 索引）+ Jev 精排（固定候选数、位置校准） | 能召回相关记忆，且顺序扰动测试显示结果稳定 |
| **5. 整理层** | `/dream` 入口 + 四道门 + 四阶段 | 手动跑一次能正确合并重复、标出冲突、修剪索引 |

**关键**：**步骤 1 完全不依赖任何模型**，可以先把端到端链路跑通。这样不会卡在 API 配额或环境搭建上。

---

## 12. 待决策

| # | 问题 | 状态 | 倾向 |
| --- | --- | --- | --- |
| 1 | 判断层后端 | ✅ **已定**：v0.1 只适配 Jev，Kev 留接口 | —— |
| 2 | 存储模型：taxonomy path ↔ entry 一对一（key-value 风格），还是一事实一文件（append-only 风格） | ⏳ 待定 | **前者 + 打补丁**，因为状态机更好挂 |
| 3 | 记忆仓库与 WorkBuddy 现有 memory（`~/.workbuddy/MEMORY.md` + workspace 每日日志）的关系：替代 / 共存 / 纳管 | ⏳ 待定 | **不定会双写打架**，需先明确 |

---

## 附：本设计引用的源码与来源

| 来源 | 类型 | 核实 |
| --- | --- | --- |
| `zhangfengcdt/memoir` 的 `plugins/claude-code/` 全目录 | 源码深读 | ✅ 本地浅克隆，逐文件读 |
| 其中 `hooks/prompts/stop_capture.tmpl` / `hooks/stop.sh` / `hooks/session-start.sh` / `skills/memory-recall/SKILL.md` / `commands/{recall,remember,status,sync,ui,onboard}.md` | 源码 | ✅ 全文读过 |
| `fonlan/gitmemo` 的 `SKILL.md`（Entry Format、命令契约、CJK 写入避坑） | 源码 | ✅ 全文读过 |
| Claude Code Auto-Dream 的四道门 / 四阶段 / 三层层级 / 安全约束 | 公开设计文档 | ✅ 多源检索一致（claude-wiki.com、百度百科、多篇分析） |
| Auto-Dream 的理论出处（Sleep-time Compute，arXiv:2504.13171） | 论文引用 | ⚠️ 二手转述，未读原文 |
| Kev 的 token/延迟/精度约束 | 官方 README | ✅ 亲自抓取 |
| Jev 失效模式（顺序敏感 / IIA 违背 / confidence 语义） | Archer Hume 逆向分析 | ✅ 亲自抓取 |
