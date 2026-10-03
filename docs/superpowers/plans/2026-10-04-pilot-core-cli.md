# gitev 先导原型实施路线与第一阶段任务

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 从零交付可重放、可评测的单用户记忆核心与 CLI，覆盖编程和生活；本文件详细安排第一个独立切片：不依赖模型的事件、事实及维护操作输入契约。

**Architecture:** Python 核心库负责维护语义与本地 Git 存储，CLI 只负责解析和输出。规则、Responses LLM 与 Jev 共用候选检索和执行器。先证明确定性行为，再接模型，最后扩大人工复核的数据与实验。

**Tech Stack:** Python 3.11+；第一切片只用标准库 `dataclasses`、`enum`、`datetime` 和 `unittest`。后续 CLI 使用 `argparse`，存储调用 Git；模型 HTTP 客户端在适配切片中结合真实服务选定。

---

## 交付边界与基线

- 仓库：`/Users/huguangyao.1/JD2026-1/gitev`，设计基线 `6316dfe`。
- 权威设计：[先导原型设计](../specs/2026-09-28-pilot-core-cli-design.md)。本路线不改变 60 条意图、24 个生命周期序列以及分域、分轨道的范围。
- 用户提供 LLM 服务与 Jev Key；通用 LLM 只支持 Responses API。
- 本次只有规划文档。任务复选框表示未来实施步骤，代码块是计划内容，不表示文件已创建或测试已运行。
- 不在 `kf-intelligent` 中开发。首次编码建立 `codex/` 分支及适合该仓库的隔离 checkout；本次文档沿用已承载设计的干净 checkout。
- 本文件给出完整先导依赖路线和第一切片的可执行任务。其余切片在前置切片通过、接口得到实际验证后分别形成详细计划，避免提前写出依赖尚未存在的完整实现。

## 依赖路线与阶段验收

| 切片 | 工作 | 通过后可验证的结果 | 依赖 |
| --- | --- | --- | --- |
| P1 契约与小场景 | 输入身份、作用域、证据片段和操作；编程/生活各一个短场景 | 无凭据即可验证合法输入，拒绝错主体、错作用域和不存在的来源 | 无 |
| P2 确定性维护引擎 | 接收记录、提案、七类操作、完整提交、幂等及恢复 | 可重放新增→待审→取代→撤回；重复和崩溃不产生半份事实 | P1 |
| P3 可用 CLI | init/ingest/recall/review/inspect/history；JSON 与退出码 | 以结构化事件操作独立仓库，查看当前事实、冲突、历史及失败 | P2 |
| P4 模型适配 | Responses 抽取和 LLM 判断、Jev 判断、规则基线、调用日志 | 小批真实输入三种判断可比较，失败保留且不会误写 | P2；小批通过 P3 操作 |
| P5 场景与评测 | 60 意图、24 序列；人工复核；隔离与端到端、自动与人工辅助轨道 | eval 调用同一个核心，逐步断言且不给模型未来或答案 | P2–P4 |
| P6 先导结论 | 小批预算→全先导运行→汇总与失败解释 | 域/操作/轨道结果及成本可重算，明确是否值得扩大实验 | P5；真实凭据与人工复核 |

P2 的确定性测试使用显式提案及测试策略验证执行器；它不等于未经校准的模型获准自动写入。真实语义操作没有校准策略时仍待审。后续七个命令由同一核心提供；eval 在 P5 接入。

跨切片验收必须覆盖：pending 不隐藏旧事实、同 ID 不同内容失败、完整取代、陈旧提案不能应用、原来源撤回后重放不能恢复、检查失败不误判过期、个人和项目不互相覆盖、历史时间不凭空推断。Jev 是否领先由结果决定。

## 文件职责地图

| 文件或目录 | 职责 | 切片 |
| --- | --- | --- |
| `src/gitev/contracts.py` | 输入契约、证据校验与操作标识 | P1 |
| `tests/test_contracts.py` | 作用域、证据及输入边界 | P1 |
| `data/pilot/drafts/` | 可见事件与人工标签分开，草稿带复核状态 | P1 / P5 |
| `src/gitev/storage.py` | 已提交快照、接收记录、隔离准备与 Git 原子发布 | P2 |
| `src/gitev/maintenance.py` | 记忆和提案模型、合法转移、事件幂等、review | P2 |
| `src/gitev/retrieval.py` | 主体/作用域过滤、稳定候选、当前/历史视图 | P2 |
| `src/gitev/inspection.py` | 显式时刻与可控环境证据检查 | P2 |
| `src/gitev/cli.py`、`src/gitev/__main__.py` | 参数、格式、stdout/stderr 与退出码 | P3 |
| `src/gitev/config.py` | 按需读取环境配置，缺失时明确失败 | P4 |
| `src/gitev/models/responses.py` | Responses 请求、完成状态、文本和结构解析 | P4 |
| `src/gitev/models/jev.py` | 按真实 Jev 协议编码问题、解析概率 | P4 |
| `src/gitev/models/rules.py` | 可复现规则基线及其支持边界 | P4 |
| `src/gitev/extraction.py`、`src/gitev/judgment.py` | 抽取与判断的业务输入输出；统一适配模型 | P4 |
| `src/gitev/call_log.py` | 非敏感请求/响应、耗时、用量及失败记录 | P4 |
| `src/gitev/evaluation.py` | 隔离运行、可见输入投影、逐步断言与指标分母 | P5 |
| `tests/test_storage.py`、`tests/test_maintenance.py`、`tests/test_retrieval.py`、`tests/test_inspection.py` | 确定性行为与失败恢复 | P2 |
| `tests/test_cli.py`、`tests/test_responses.py`、`tests/test_jev.py`、`tests/test_evaluation.py` | 命令、协议与评测隔离 | P3–P5 |
| `data/pilot/intent.jsonl`、`data/pilot/lifecycle.jsonl`、`data/pilot/annotations.jsonl` | 场景、预期状态与标签来源 | P5 |
| `docs/reports/pilot.md` | 先导结果、错误解释、费用与后续决定 | P6 |

这是一份职责地图，不在第一切片预建全部空模块。

## 模型接入与实验安排

1. LLM 配置拟用 `GITEV_LLM_BASE_URL`、`GITEV_LLM_MODEL`、`GITEV_LLM_API_KEY`；Jev 凭据沿用 `TYPESAFE_API_KEY`。命令只在要调用该服务时读取配置。
2. Responses 先做单次非流式文本请求；发送本次完整上下文，返回状态须完成，文本须通过本地结构校验。拒绝、未完成、限流、超时、非法 JSON 都保留为失败。结构化输出参数等可选能力在真实服务上探测，不以“OpenAI 兼容”推断全部参数支持。
3. Jev 接入时核对官方接口与账号可用能力，保存真实样例及返回概率；不凭模型名称编造 SDK 调用或将概率直接当正确率。
4. 先用编程和生活各两个场景确认可访问性、协议、日志和单位费用，再扩到先导。运行前生成预计调用量与费用范围，按提供服务的实际价格计算；缺用量不能记作零成本。
5. 确定性行为测试使用可控响应。真实服务兼容性和模型成绩另留实测证据，模拟测试不能替代它们。
6. 人工只需先复核两个短场景的事实变化和预期结果；在引入全部先导样例前再复核并记录分歧。无人确认时草稿可以跑工程诊断，不能报告人工验证数据成绩。

## CLI 的统一约定

P3 采用全局 `--repo` 与 `--json`；`ingest` 接收 `--input` JSON 文件、`--format structured|text`、`--judge rules|llm|jev`，诊断时使用 `--propose-only`。事件 ID 可以在输入中提供；生成的 ID 在接收记录持久化后返回，重试复用原 ID。

`review` 采用 `--proposal-id` 和 `--action show|approve|reject`；`history` 按 `--memory-id` 或 `--event-id` 查询；`inspect` 使用 `--now` 和 `--evidence`；`recall` 使用 `--query`、`--scope personal|project`、`--project-id`、`--limit`。`eval` 使用 `--dataset`、`--output`、`--judge`，并显式选择输入与辅助轨道。具体输入 JSON schema 由已验证的核心契约给出。

退出码约定：`0` 正常完成（含待审）；`2` 参数/配置/输入错误；`3` 模型调用或响应失败；`4` 版本、输入身份或未识别工作区修改冲突；`5` 存储/提交失败。混合结果中只要含失败即返回相应非零码，stdout 仍保存全部逐项结果；多种失败时优先级为存储、版本、模型、输入。详细失败类型保留在 JSON 中。

## 第一切片：输入契约库

本机默认 `python3` 为 3.9.6，已有 `python3.11` 可用。以下命令显式使用 `python3.11`，避免把解释器版本错误当成业务失败。

第一切片交付一个可导入并执行校验的核心契约库，以及两个标注草稿。它不开放摄入或自动写入接口。P2 在契约上增加存储对象和状态转移，不要求 P1 提前定义全部历史与运行日志结构。

### Task 1：把两个短场景固定为人工复核草稿

**Files:**
- Create: `data/pilot/drafts/coding-project-preference.json`
- Create: `data/pilot/drafts/daily-moving.json`

- [ ] **Step 1：写入编程场景。**

```json
{
  "scenario_id": "coding-project-preference",
  "domain": "coding",
  "annotation_status": "draft",
  "reviewer": null,
  "events": [
    {"event_id": "c1", "source_id": "coding:1", "scope": {"kind": "personal"}, "subject": "user", "speaker": "user", "occurred_at": "2026-10-04T09:00:00+08:00", "text": "我平时习惯用 pytest 写测试。"},
    {"event_id": "c2", "source_id": "coding:2", "scope": {"kind": "project", "project_id": "project-a"}, "subject": "user", "speaker": "user", "occurred_at": "2026-10-04T09:10:00+08:00", "text": "这个项目统一用 unittest。"}
  ],
  "oracle": {
    "current_personal": ["个人默认使用 pytest"],
    "current_project": ["project-a 使用 unittest"],
    "forbidden": ["把个人 pytest 默认标记为失效", "在个人查询中默认混入 project-a 约定"]
  }
}
```

- [ ] **Step 2：写入生活场景。**

```json
{
  "scenario_id": "daily-moving",
  "domain": "daily",
  "annotation_status": "draft",
  "reviewer": null,
  "events": [
    {"event_id": "d1", "source_id": "daily:1", "scope": {"kind": "personal"}, "subject": "user", "speaker": "user", "occurred_at": "2026-10-04T10:00:00+08:00", "text": "我住在杭州，平时喜欢清淡饮食。"},
    {"event_id": "d2", "source_id": "daily:2", "scope": {"kind": "personal"}, "subject": "user", "speaker": "user", "occurred_at": "2026-10-04T10:10:00+08:00", "text": "我已经搬到上海，以后住在上海了。"}
  ],
  "oracle": {
    "current": ["现居上海", "偏好清淡饮食"],
    "history": ["原居杭州"],
    "forbidden": ["普通查询返回杭州为当前住址", "因为搬家而删除饮食偏好"]
  }
}
```

`oracle` 只由评测器读取；未来抽取和判断请求从 `events` 投影生成，绝不直接发送整个场景对象。此处两个草稿只是场景种子，不额外增加 60/24 的最终计数。受时间影响的文本解释按事件时间进行，不按运行机器当天解释。

- [ ] **Step 3：检查 JSON 有效，人工核对文本与预期边界。**

```bash
python3.11 -m json.tool data/pilot/drafts/coding-project-preference.json
python3.11 -m json.tool data/pilot/drafts/daily-moving.json
```

Expected: 两个命令均退出 `0`，输出合法 JSON。人工确认前保留 `annotation_status: draft` 与空 reviewer；复核时填写实际人员及复核记录，不把开发者跑通测试当成独立标注。

- [ ] **Step 4：只提交场景草稿。**

```bash
git diff --check
git add data/pilot/drafts/coding-project-preference.json data/pilot/drafts/daily-moving.json
git commit -m "docs: 用双场景固定作用域与独立事实的验收边界"
```

### Task 2：建立契约与证据行为测试

**Files:**
- Create: `src/gitev/__init__.py`
- Create: `src/gitev/contracts.py`
- Test: `tests/test_contracts.py`

- [ ] **Step 1：创建下面的行为测试。**

```python
# tests/test_contracts.py
import unittest
from datetime import datetime, timezone

from gitev.contracts import (
    ContractError, Domain, Event, Evidence, Fact, Operation, Scope,
    validate_fact,
)


class ContractsTest(unittest.TestCase):
    def event(self, **changes):
        values = dict(
            event_id="e1", source_id="s1", scope=Scope("personal"),
            domain=Domain.DAILY, subject="user", speaker="user",
            text="我住在杭州，平时喜欢清淡饮食。",
            occurred_at=datetime(2026, 10, 4, tzinfo=timezone.utc),
        )
        values.update(changes)
        return Event(**values)

    def fact(self, event, **changes):
        values = dict(
            scope=event.scope, subject="user", text="现居杭州",
            evidence=Evidence(event.event_id, event.source_id, 0, 6),
        )
        values.update(changes)
        return Fact(**values)

    def test_personal_scope_cannot_carry_project_id(self):
        with self.assertRaises(ContractError):
            Scope("personal", "project-a")

    def test_project_scope_requires_project_id(self):
        with self.assertRaises(ContractError):
            Scope("project")

    def test_valid_atomic_fact_preserves_source_span(self):
        event = self.event()
        fact = self.fact(event)
        validate_fact(event, fact)
        self.assertEqual(event.text[fact.evidence.start:fact.evidence.end], "我住在杭州，")
        self.assertEqual(fact.text, "现居杭州")

    def test_fact_cannot_use_another_event_or_source(self):
        event = self.event()
        for evidence in (Evidence("e2", "s1", 0, 6), Evidence("e1", "s2", 0, 6)):
            with self.subTest(evidence=evidence), self.assertRaises(ContractError):
                validate_fact(event, self.fact(event, evidence=evidence))

    def test_fact_cannot_cross_subject_or_scope(self):
        event = self.event()
        for changes in ({"subject": "friend"}, {"scope": Scope("project", "a")}):
            with self.subTest(changes=changes), self.assertRaises(ContractError):
                validate_fact(event, self.fact(event, **changes))

    def test_evidence_must_reference_nonempty_existing_text(self):
        event = self.event()
        for start, end in ((0, 0), (-1, 2), (0, 999), (True, 2)):
            with self.subTest(span=(start, end)), self.assertRaises(ContractError):
                fact = self.fact(event, evidence=Evidence("e1", "s1", start, end))
                validate_fact(event, fact)

    def test_event_time_requires_timezone(self):
        with self.assertRaises(ContractError):
            self.event(occurred_at=datetime(2026, 10, 4))

    def test_empty_fact_and_invalid_validity_are_rejected(self):
        event = self.event()
        now = event.occurred_at
        for changes in ({"text": " "}, {"valid_from": now, "valid_until": now}):
            with self.subTest(changes=changes), self.assertRaises(ContractError):
                self.fact(event, **changes)

    def test_domain_and_operations_have_distinct_roles(self):
        event = self.event(domain=Domain.CODING)
        self.assertEqual(event.scope.kind, "personal")
        self.assertEqual({op.value for op in Operation}, {
            "new", "duplicate", "update", "supersede", "contradict", "retract", "expire",
        })


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2：运行测试，确认缺失核心契约使其失败。**

Run: `PYTHONPATH=src python3.11 -m unittest discover -s tests -v`

Expected: 非零退出，导入 `gitev.contracts` 失败。若已有同名文件，先检查来源，不能覆盖未识别代码。

- [ ] **Step 3：创建空的 `src/gitev/__init__.py`，并写入完整契约实现。**

```python
# src/gitev/contracts.py
from dataclasses import dataclass
from datetime import datetime
from enum import Enum


class ContractError(ValueError):
    pass


def require_text(value: str, field: str) -> None:
    if not isinstance(value, str) or not value.strip():
        raise ContractError(f"{field} must be nonempty text")


def require_time(value: datetime, field: str) -> None:
    if not isinstance(value, datetime) or value.utcoffset() is None:
        raise ContractError(f"{field} must include a timezone")


class Domain(str, Enum):
    CODING = "coding"
    DAILY = "daily"


class Operation(str, Enum):
    NEW = "new"
    DUPLICATE = "duplicate"
    UPDATE = "update"
    SUPERSEDE = "supersede"
    CONTRADICT = "contradict"
    RETRACT = "retract"
    EXPIRE = "expire"


@dataclass(frozen=True)
class Scope:
    kind: str
    project_id: str | None = None

    def __post_init__(self):
        if self.kind == "personal" and self.project_id is None:
            return
        if self.kind == "project":
            require_text(self.project_id, "project_id")
            return
        raise ContractError("scope must be personal without project_id or project with project_id")


@dataclass(frozen=True)
class Event:
    event_id: str
    source_id: str
    scope: Scope
    domain: Domain
    subject: str
    speaker: str
    text: str
    occurred_at: datetime

    def __post_init__(self):
        for field in ("event_id", "source_id", "subject", "speaker", "text"):
            require_text(getattr(self, field), field)
        if not isinstance(self.scope, Scope) or not isinstance(self.domain, Domain):
            raise ContractError("scope and domain must use validated contract types")
        require_time(self.occurred_at, "occurred_at")


@dataclass(frozen=True)
class Evidence:
    event_id: str
    source_id: str
    start: int
    end: int

    def __post_init__(self):
        require_text(self.event_id, "evidence.event_id")
        require_text(self.source_id, "evidence.source_id")
        if type(self.start) is not int or type(self.end) is not int:
            raise ContractError("evidence offsets must be integers")
        if not 0 <= self.start < self.end:
            raise ContractError("evidence must identify a nonempty span")


@dataclass(frozen=True)
class Fact:
    scope: Scope
    subject: str
    text: str
    evidence: Evidence
    valid_from: datetime | None = None
    valid_until: datetime | None = None

    def __post_init__(self):
        require_text(self.subject, "fact.subject")
        require_text(self.text, "fact.text")
        if not isinstance(self.scope, Scope) or not isinstance(self.evidence, Evidence):
            raise ContractError("fact must use validated scope and evidence")
        for field in ("valid_from", "valid_until"):
            value = getattr(self, field)
            if value is not None:
                require_time(value, field)
        if self.valid_from is not None and self.valid_until is not None:
            if self.valid_from >= self.valid_until:
                raise ContractError("valid_until must follow valid_from")


def validate_fact(event: Event, fact: Fact) -> None:
    if fact.scope != event.scope or fact.subject != event.subject:
        raise ContractError("fact must preserve event scope and subject")
    evidence = fact.evidence
    if evidence.event_id != event.event_id or evidence.source_id != event.source_id:
        raise ContractError("fact must refer to its input event and source")
    if evidence.end > len(event.text):
        raise ContractError("evidence span exceeds input text")
    if not event.text[evidence.start:evidence.end].strip():
        raise ContractError("evidence span cannot contain only whitespace")
```

此契约针对一个明确主体和范围的输入事件；对话有多个主体或范围时，抽取器必须给出各自的有来源候选，不能给原事件静默换主体。P4 在可见消息范围内验证这类候选，并保留它与原接收事件的关联；本切片只接受已经明确主体和范围的结构化事件。原子性是标签与抽取约束，不能靠字符数验证。

- [ ] **Step 4：运行同一命令，确认九个行为测试通过。**

Run: `PYTHONPATH=src python3.11 -m unittest discover -s tests -v`

Expected: `Ran 9 tests` 和 `OK`。本测试集合验证输入边界，不证明状态转移、模型或存储正确。

- [ ] **Step 5：规格符合性与代码质量检查通过后，单独提交。**

```bash
git diff --check
git add src/gitev/__init__.py src/gitev/contracts.py tests/test_contracts.py
git commit -m "feat: 固定记忆输入的作用域与来源边界"
```

## 第一阶段交接与下一份计划

P1 的验收是契约库可运行、九项输入边界通过、两个草稿可审阅。完整先导仍依赖 P2–P6；本切片不宣称已完成记忆维护。

P2 详细计划必须先解决一个具体工程选择：如何隔离准备整批变更并在一个 Git 提交后发布给读者。以临时 index / Git 对象准备、引用 compare-and-swap 与已提交快照读取作为优先验证方向；不能先逐文件改工作区，再把普通 commit 当成完整原子发布。对于可供用户查看的工作区，同步过程的中断必须可恢复，且不能误认用户手工修改；验证该协议后才能固定实现。

P2 计划需要包含故障点的可运行测试：发布前中断、发布后返回前中断、模型调用期间版本变化、review 期间目标变化、同事件失败重试及原来源撤回后重放。模型调用在写锁外；调用返回后重新检查版本；一次事件内相互依赖的提案不能按数组顺序随意应用。

实施采用新实现子代理处理独立切片；完成后先检查规格、再检查质量，相关测试通过即中文提交。存储与执行接口稳定后，模型适配和数据草稿可并行；共享执行器的改动保持顺序。分阶段展示可运行结果、测试证据和未验证范围。

原先 4–6 人日是先导工作量目标，包含真实服务可用与有人复核标签的前提。第一轮存储故障测试、模型小批及标注复核后重估剩余投入；不能用生成代码的速度推算正式评测和论文工期。

## 计划自查

- 第一切片没有模型、CLI、存储的空壳模块；所有导入类型都在本任务定义。
- 测试验证的是来源、时间和作用域行为，数据草稿明确未复核。
- 完整设计的维护、review、inspect、当前/历史、调用日志、评测隔离和预算均映射到路线中的切片。
- 其余切片仍需各自的详细实施计划，不能直接把职责地图当成可执行代码任务。
