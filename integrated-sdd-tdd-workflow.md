# SDD + TDD 融合工作流设计

## 1. 定位

本工作流用于长期维护、多人协作、业务行为需要稳定交付的工程项目。它吸收 OpenSpec、Matt-skills、Superpowers 三类工作流的优势，但不直接拼接三者命令，而是围绕 `final-workflow.md` 的核心分层重新设计：

- SDD 负责业务意图、外部契约、关键约束和交付边界。
- TDD 负责行为切片内的小步实现、设计反馈和回归保护。
- Scenario 是 Spec 与 Test 的桥接单位。
- Test 是 Spec 的可执行证据，但不是外部业务契约的默认事实来源。
- Escape Hatch 是从 TDD 反馈回到 SDD 的轻量通道。

工作流目标不是最大化文档数量，也不是把每个测试和每条文档机械绑定，而是让 Spec、Test、Code 在行为语义上保持一致。

## 2. 继承与取舍

### 2.1 从 OpenSpec 继承

- Change folder：一次变更有独立目录，便于并行、审查、归档。
- Delta Spec：只描述本次变更对外部行为的新增、修改、移除。
- Archive：完成后把变更沉淀到主 Spec 和历史归档。
- 状态可查询：状态由文件、checkbox、机器状态共同表达，而不是只靠聊天上下文。

不继承的部分：

- 不保留过多命令分支。
- 不强制 proposal、design、tasks 拆成过多独立 artifact。
- 不让 `apply` 退化为“按任务列表直接写代码”，而是在每个行为切片内执行 TDD。

### 2.2 从 Matt-skills 继承

- Grill / clarify：实现前必须先澄清问题、范围、非目标和领域语言。
- CONTEXT / ADR：用轻量领域语言和少量架构决策减少后续沟通成本。
- Tracer bullet / vertical slice：任务拆分按端到端行为增量，而不是技术层横切。
- Issue 可导出：计划成熟后可拆为独立 issue，但 issue tracker 不作为核心事实来源。

不继承的部分：

- 不把 PRD、issue、triage 作为每次变更的强制核心路径。
- 不依赖外部 issue label 状态机来表达 Spec/Test/Code 的一致性。

### 2.3 从 Superpowers 继承

- TDD 铁律：新行为和 bug 修复优先写失败测试，再写最小实现。
- Subagent execution：独立切片可用多个 agent 并行实现。
- 双层 review：先查 Spec compliance，再查 code quality。
- Evidence before claims：完成声明必须有测试、review 或验证证据。

不继承的部分：

- 不要求每个小功能都写长篇 design doc 和超细 step-by-step plan。
- 不把 plan 写成易过期的逐行代码脚本。
- 不把每个门禁都变成强硬阻塞，治理强度按项目风险分级。

## 3. 设计原则

### 3.1 命令精简，但认知台阶不能合并

工作流保留 5 个核心 Skill 命令：

```text
/flow:clarify -> /flow:spec -> /flow:plan -> /flow:apply -> /flow:close
```

另有 2 个辅助 Skill：

```text
/flow:escape
/flow:review
```

`clarify`、`spec`、`plan` 必须分开。需求澄清、外部契约定义、行为切片拆分是三个不同抽象层级，合并后容易产生认知间隙。

### 3.2 Skill 场景优先，不设计成 CLI 参数

命令面向 AI agent 与用户协作场景，而不是命令行工具参数。例如实现阶段不使用：

```text
/flow:apply --parallel 4
```

而使用自然场景：

```text
/flow:apply next slice
/flow:apply resume current slice
/flow:apply delegate independent slices
/flow:apply finish blocked slice
```

并行度、可并行切片、写入边界、review 顺序由 Skill 根据 `plan.md` 和 `state.json` 判断。

### 3.3 Soft 依赖优先，硬门禁后移

核心命令在缺少上游产物时应优先给出补全路径，而不是直接失败：

- 用户直接 `/flow:apply`：若没有 `plan.md`，先提出生成最小单 Slice；若没有 `spec.md`，提示该实现只能作为轻量 TDD，不可归档进主 Spec。
- 用户直接 `/flow:spec`：若没有 `brief.md`，先从当前对话生成轻量 brief 草稿，并标记仍需确认。
- 用户直接 `/flow:close`：只有最终一致性校验需要硬门禁；缺失关键产物时应列出缺口和推荐补齐命令。

这样保留低启动成本，同时把真正影响交付可信度的约束集中到 `/flow:close`。

### 3.4 文档少，但层级清晰

一次 change 默认只有 3 份人读文档和 1 份机器状态：

```text
flow/changes/<change>/
  brief.md
  spec.md
  plan.md
  state.json
```

三份文档分别对应三个抽象层级：

- `brief.md`：为什么做，做什么，不做什么，已澄清的语境。
- `spec.md`：外部可观察行为、Scenario、契约 delta。
- `plan.md`：行为切片、测试策略、依赖关系、并行边界。

不把内部私有函数、临时实现判断、逐行 coding 指令写进 Spec。

### 3.5 Scenario 是桥接单位

Scenario 同时服务三件事：

- 在 SDD 层表达外部可验证行为。
- 在 TDD 层派生验收、集成、契约或单元测试策略。
- 在 Verify 层检查 Spec/Test/Code 是否语义一致。

重要 Scenario 应有稳定 ID，例如：

```text
auth.login.success
auth.login.invalid-password
billing.invoice.retry-timeout
```

关键验收、集成、契约测试建议显式引用 Scenario ID，例如：

```text
@scenario auth.login.success
```

普通单元测试不强制绑定 Scenario ID，避免把内部实现细节伪装成业务契约。

### 3.6 切片按行为纵切

每个 Slice 必须是一个可独立验证的行为增量，而不是“先改数据库、再改 API、再改 UI”的水平层切分。

合格 Slice 应满足：

- 对应一个或多个 Scenario。
- 有可观察结果。
- 能独立测试。
- 有明确依赖。
- 有写入范围。
- 能说明是否可并行。

每个 Slice 还应标注执行类型：

- `AFK`：上下文清楚、验收明确、可由子 agent 独立完成。
- `HITL`：需要人类判断、产品取舍、外部确认或高风险操作。

### 3.7 TDD 只在切片内执行

Spec 不列出所有单元测试。Plan 只定义测试策略和关键证据，不提前批量写满测试清单。

每个 Slice 内执行：

```text
Red -> Green -> Refactor -> Review -> Evidence
```

### 3.8 Escape 是一等能力

实现中如果发现 Scenario 错误、契约缺失、接口语义需要改变、范围明显溢出，必须触发 `/flow:escape`，而不是让测试或代码悄悄改写业务契约。

Escape 除写入 `state.json` 外，L1 以后建议同步写入 append-only 的 `escapes.log`，用于审计和复盘。

## 4. 目录与产物

### 4.1 推荐目录

```text
flow/
  specs/
    <capability>.md
  changes/
    <change>/
      brief.md
      spec.md
      plan.md
      state.json
      escapes.log              # L1+ 可选，append-only
      evidence/                # L1+ 可选，slice/review 证据快照
        <slice-id>/
          notes.md
    archive/
      YYYY-MM-DD-<change>/
        brief.md
        spec.md
        plan.md
        state.json
        escapes.log
        evidence/

CONTEXT.md
docs/adr/
```

### 4.2 产物职责

| 产物 | 事实来源职责 | 主要创建命令 | 是否必须 |
|---|---|---|---|
| `brief.md` | 意图、范围、非目标、澄清记录、开放问题 | `/flow:clarify` | 必须 |
| `spec.md` | Scenario、验收语义、外部契约 delta | `/flow:spec` | 必须 |
| `plan.md` | 行为切片、测试策略、依赖、并行边界 | `/flow:plan` | 必须 |
| `state.json` | 机器状态、slice claim、证据索引、escape 状态 | `/flow:plan` 后持续更新 | 必须 |
| `escapes.log` | append-only 的 Escape 审计记录 | `/flow:escape` | L1+ 建议 |
| `evidence/` | 子 agent 报告、review 结论、测试输出摘要 | `/flow:apply`、`/flow:review` | L1+ 建议 |
| `flow/specs/*.md` | 系统当前外部行为的主 Spec | `/flow:close` 同步 | 建议 |
| `CONTEXT.md` | 领域语言 | `/flow:clarify` 可更新 | 可选 |
| `docs/adr/*.md` | 难逆转、非显然、有真实权衡的架构决策 | `/flow:clarify` 或 `/flow:review` 可建议 | 可选 |

## 5. 产物模板

### 5.1 `brief.md`

```markdown
# Brief: <change>

## Problem

## Goal

## Scope

## Non-Goals

## Domain Language

## Constraints

## Open Questions

## Decision Log
```

### 5.2 `spec.md`

```markdown
# Spec Delta: <change>

## Capabilities

## ADDED Requirements

### Requirement: <name>
The system SHALL ...

#### Scenario: <scenario-id>
- GIVEN ...
- WHEN ...
- THEN ...

## MODIFIED Requirements

## REMOVED Requirements

## External Contracts

## Acceptance Evidence Required
```

### 5.3 `plan.md`

```markdown
# Plan: <change>

## Strategy

## Slice Graph

## Slices

### SL1: <behavior increment>
Status: pending
Execution Mode: AFK
Scenarios: <scenario-id>
Dependencies: none
Parallel Group: A
Estimated TDD Cycles: 3
Write Scope:
- src/...
- tests/...

Test Strategy:
- Acceptance:
- Integration:
- Contract:
- Unit:

Tasks:
- [ ] Write failing test for ...
- [ ] Implement minimal behavior
- [ ] Refactor under green tests
- [ ] Record evidence

## Parallelization Rules

## Escape Policy

## Verification Checklist
```

### 5.4 `state.json`

`state.json` 是机器状态，不应成为人类主要阅读入口。

```json
{
  "change": "add-auth-login",
  "status": "planning",
  "current_slice": null,
  "slices": {
    "SL1": {
      "status": "pending",
      "owner": null,
      "execution_mode": "AFK",
      "estimated_cycles": 3,
      "scenarios": ["auth.login.success"],
      "write_scope": ["src/auth/**", "tests/auth/**"],
      "evidence": []
    }
  },
  "escapes": [],
  "escape_log": "escapes.log",
  "reviews": [],
  "updated_at": "YYYY-MM-DDTHH:mm:ssZ"
}
```

## 6. 命令清单

### 6.1 `/flow:clarify`

**调用时机**

- 用户提出新功能、行为变更、复杂 bug、性能目标、接口调整等模糊需求时。
- 已有需求但范围、目标、非目标、业务语言不清楚时。
- 实现中 Escape 过多，需要退回需求澄清时。

**输入**

- 用户原始想法或问题描述。
- 现有代码结构、相关测试、已有 Spec。
- `CONTEXT.md`、`docs/adr/`、历史 change 或 issue。

**行为**

- 探索代码和现有文档，能从仓库回答的问题不重复问用户。
- 一次只问一个关键问题，优先解决会影响范围和契约的问题。
- 澄清 Problem、Goal、Scope、Non-Goals、Constraints。
- 识别领域术语冲突，必要时更新 `CONTEXT.md`。
- 识别是否适合完整流程、轻量流程或纯 TDD。
- 对高风险或难逆转决策建议 ADR，但不默认创建。
- 如果用户跳过澄清直接进入 `/flow:spec`，应从当前上下文生成轻量 brief，并标记需要用户确认的假设。

**产物**

- 创建或更新 `flow/changes/<change>/brief.md`。
- 初始化 `state.json`，change 状态为 `clarifying` 或 `draft`。
- 可选更新 `CONTEXT.md`。

**关键取舍**

`clarify` 不写正式 Scenario，不拆任务，不进入实现。它只解决“我们到底要解决什么问题，以及不解决什么问题”。

### 6.2 `/flow:spec`

**调用时机**

- `brief.md` 的目标、范围和非目标已足够稳定。
- 需要定义新增、修改或移除的外部行为。
- Bug 修复改变外部行为，或发现现有 Spec 缺失。

**输入**

- `brief.md`。
- 主 Spec：`flow/specs/*.md`。
- 相关代码和现有测试。
- 用户在澄清阶段确认的边界。

**行为**

- 把需求转成行为契约，而不是实现计划。
- 定义 Capability、Requirement、Scenario。
- 为 Scenario 分配稳定 ID。
- 标注 ADDED、MODIFIED、REMOVED。
- 记录外部 API、数据模型、事件、错误语义、权限、安全、兼容性、性能承诺等契约变化。
- 明确 acceptance evidence 类型，但不预写所有测试。

**产物**

- 创建或更新 `flow/changes/<change>/spec.md`。
- 更新 `state.json` 中 scenario 索引。
- change 状态推进到 `specified`。

**关键取舍**

`spec` 不应写私有函数、内部目录结构、具体缓存策略、mock 策略。若实现细节不影响外部行为，放到 `plan.md` 或 TDD 内处理。

### 6.3 `/flow:plan`

**调用时机**

- `spec.md` 的主要 Scenario 和外部契约已稳定。
- 准备进入实现前。
- 需要把变更拆成可实现、可测试、可并行的行为切片。

**输入**

- `brief.md`。
- `spec.md`。
- 现有代码结构、测试结构、构建命令。
- 相关 ADR 和领域语言。

**行为**

- 按 Scenario 和风险拆分行为 Slice。
- 为每个 Slice 定义依赖、写入范围、测试策略、完成证据。
- 为每个 Slice 标注 `AFK` 或 `HITL`。
- 为每个 Slice 粗估 TDD cycle 数；超过 10 个 cycle 应提示拆分。
- 标记哪些 Slice 可以并行，哪些必须串行。
- 给出每个 Slice 的 TDD 入口：先写哪个失败测试，验证什么行为。
- 定义 Escape 策略和触发阈值。
- 不写过细逐行代码计划，除非是高风险迁移或复杂协议。

**产物**

- 创建或更新 `flow/changes/<change>/plan.md`。
- 初始化或更新 `state.json` 中 slice 状态。
- change 状态推进到 `ready`。

**关键取舍**

`plan` 的目标是让实现不会偏离 Spec，并让并行执行可控。它不是 Superpowers 式完整代码脚本，也不是 OpenSpec 式普通 checkbox 列表。

### 6.4 `/flow:apply`

**调用时机**

- `plan.md` 已存在，change 状态为 `ready` 或 `implementing`。
- 用户希望开始、继续、恢复或并行实现行为切片。

**推荐调用场景**

```text
/flow:apply
/flow:apply next slice
/flow:apply resume current slice
/flow:apply implement ready slices
/flow:apply delegate independent slices
/flow:apply finish blocked slice
/flow:apply run AFK slices
/flow:apply work HITL slice
```

**输入**

- `brief.md`、`spec.md`、`plan.md`。
- `state.json`。
- 现有代码、测试、构建命令。
- 用户对当前执行模式的自然语言意图。

**行为**

- 读取状态，判断当前是否有进行中或阻塞 Slice。
- 选择下一个 Slice 或一组可并行 Slice。
- 对每个 Slice 执行 TDD：

```text
Red: 写一个针对当前行为的失败测试
Green: 写最少代码让测试通过
Refactor: 在全绿状态下改善结构
Review: 先做 Spec compliance，再做 code quality
Evidence: 记录测试和审查证据
```

- 如果用户要求并行，Skill 根据写入范围和依赖判断是否可以委派多个 agent。
- `AFK` Slice 可委派子 agent；`HITL` Slice 留在主会话执行，或先向用户确认关键决策后再降级为 `AFK`。
- 子 agent 必须拿到明确上下文：Slice、Scenario、写入范围、测试策略、禁止越界项。
- 子 agent 不应修改其他 Slice 的写入范围，不应自行改变外部契约。
- 关键行为测试应引用对应 Scenario ID，例如 `@scenario auth.login.success`。
- 完成一个 Slice 后更新 `plan.md` checkbox、`state.json`，L1+ 可写入 `evidence/<slice-id>/notes.md`。

**产物**

- 代码和测试变更。
- `plan.md` 中 Slice / Task checkbox 更新。
- `state.json` 中 Slice 状态、owner、evidence 更新。
- 必要时生成 Escape 记录。

**关键取舍**

`apply` 不是“执行所有任务直到结束”的黑盒。它是按行为切片推进的 TDD Skill。并行是内部决策，不靠 CLI 参数控制。

### 6.5 `/flow:escape`

**调用时机**

- TDD 中发现 Scenario 语义错误、歧义或矛盾。
- 测试暴露 Spec 未覆盖的外部边界。
- 实现显示外部接口或契约需要调整。
- Bug 修复会改变外部行为或验收标准。
- Slice 实现成本明显超出预期，需要重新切片或缩小范围。

**输入**

- 当前 Slice。
- 当前失败测试或实现证据。
- 相关 Scenario 和 Spec。
- 问题类型。

**行为**

- 暂停当前 TDD 小循环。
- 记录 Escape 类型：

```text
[escape:spec-error]
[escape:scenario-missing]
[escape:better-interface]
[escape:scope-overflow]
```

- 修改最小必要范围的 `brief.md`、`spec.md` 或 `plan.md`。
- 更新测试策略。
- 关闭 Escape 后恢复 `/flow:apply`。
- L1+ 同步 append 到 `escapes.log`，不得删除历史 Escape 记录。

**产物**

- `spec.md` 或 `plan.md` 的局部更新。
- `state.json` 中 escape 记录。
- L1+ 的 `escapes.log` 审计记录。
- 当前 Slice 的状态从 `blocked` 或 `escaped` 恢复到 `pending` / `red` / `green`。

**关键取舍**

Escape 是正式回流通道，不是失败。它避免测试和代码悄悄成为业务契约的事实来源。

### 6.6 `/flow:review`

**调用时机**

- `spec.md` 写完后，检查 Scenario 和契约质量。
- `plan.md` 写完后，检查 Slice 粒度、依赖和并行边界。
- 一个或多个 Slice 完成后，检查 Spec compliance 或 code quality。
- `/flow:close` 前做最终一致性校验。

**输入**

- review 对象：`brief`、`spec`、`plan`、`slice`、`change`。
- 对应文件和代码 diff。
- 测试证据。

**行为**

- 按对象选择审查重点：

| 对象 | 审查重点 |
|---|---|
| `brief` | 问题、目标、范围、非目标是否清晰 |
| `spec` | Scenario 是否可验证，契约是否完整，是否混入内部实现 |
| `plan` | Slice 是否纵切，依赖是否正确，写入范围是否可并行 |
| `slice` | 是否满足对应 Scenario，是否有测试证据 |
| `change` | Spec/Test/Code 是否语义一致 |

- 输出 Critical、Warning、Suggestion。
- Critical 必须处理或显式接受风险后才能 close。

**产物**

- review 报告。
- `state.json` 中 review 记录。
- 可能触发 `/flow:escape`。

**关键取舍**

`review` 是辅助能力，不应替代 `/flow:close`。它用于提前发现问题，降低最终验证成本。

### 6.7 `/flow:close`

**调用时机**

- 所有计划内 Slice 已完成。
- Escape 已关闭。
- 关键测试和 review 已有证据。
- 准备合并、归档或交付。

**输入**

- `brief.md`、`spec.md`、`plan.md`、`state.json`。
- 当前代码 diff。
- 测试结果、review 结果。
- 主 Spec。

**推荐调用场景**

```text
/flow:close
/flow:close check only
/flow:close strict
/flow:close archive ready changes
```

**行为**

- 执行结构化校验：
  - 每个 Requirement 至少有一个 Scenario。
  - Scenario 使用稳定 ID，并符合 Given / When / Then 语义。
  - Delta 标记没有互相冲突。
- 执行 Spec/Test/Code 一致性校验：
  - Spec 是否仍表达真实业务意图。
  - 主要 Scenario 是否有自动化测试或明确证据。
  - 关键 Scenario 的测试是否引用 Scenario ID。
  - 外部接口、错误语义、数据模型、权限、安全、性能承诺是否一致。
  - 所有 Escape 是否闭环。
  - TDD 中发现的新业务语义是否回填 Spec。
  - Spec 是否没有混入内部实现细节。
- 同步 `spec.md` delta 到 `flow/specs/*.md`。
- 归档 change。
- 输出完成证据和剩余风险。
- `check only` 只生成校验报告，不同步和归档。
- `strict` 将 Warning 也视为阻塞。
- `archive ready changes` 可批量归档已完成且无 Critical 的多个 change。

**产物**

- 更新后的 `flow/specs/*.md`。
- `flow/changes/archive/YYYY-MM-DD-<change>/`。
- close report。
- `state.json` 状态为 `archived`。
- 可选 `verify-report.md`。

**关键取舍**

`close` 不要求每个 Scenario 唯一对应一个测试，也不要求每个 unit test 绑定 Scenario。它只要求外部行为无漂移、关键约束无遗漏、测试证据可信。

## 7. 标准产物流

```text
用户想法 / bug / change request
        |
        v
/flow:clarify
        |
        v
brief.md
        |
        v
/flow:spec
        |
        v
spec.md
        |
        v
/flow:plan
        |
        v
plan.md + state.json
        |
        v
/flow:apply
        |
        +-- Slice 内 TDD: Red -> Green -> Refactor
        |
        +-- /flow:escape -> spec/plan 局部修正 -> 回到 apply
        |
        v
代码 + 测试 + evidence
        |
        v
/flow:close
        |
        v
flow/specs/*.md + archive
```

### 7.1 文件生命周期

| 文件 | 创建时机 | 修改主体 | 归档/保留策略 |
|---|---|---|---|
| `brief.md` | `/flow:clarify` | `/flow:clarify`、必要时 `/flow:escape` | 随 change 归档 |
| `spec.md` | `/flow:spec` | `/flow:spec`、`/flow:escape` | delta 同步到主 Spec，原文件随 change 归档 |
| `plan.md` | `/flow:plan` | `/flow:plan`、`/flow:apply`、`/flow:escape` | 随 change 归档 |
| `state.json` | `/flow:plan` 或轻量 apply fallback | 所有 `/flow:*` 命令 | 随 change 归档，作为机器状态快照 |
| `escapes.log` | 首次 `/flow:escape` | 仅 append | 随 change 归档，历史记录不得删除 |
| `evidence/<slice-id>/notes.md` | `/flow:apply` 或 `/flow:review` | 子 agent / controller | 随 change 归档，可作为 review 和测试证据索引 |
| `flow/specs/*.md` | `/flow:close` 首次同步 capability | `/flow:close` | 项目级长期演进 |
| `CONTEXT.md` | 懒创建 | `/flow:clarify` | 项目级长期保留 |

## 8. 状态机

### 8.1 Change 状态机

```text
none
  |
  v
clarifying
  |
  v
draft
  |
  v
specified
  |
  v
ready
  |
  v
implementing
  |
  +----> escaping ----+
  |                   |
  +<------------------+
  |
  v
verifying
  |
  v
archived
```

状态说明：

| 状态 | 含义 |
|---|---|
| `clarifying` | 正在澄清需求和范围 |
| `draft` | `brief.md` 已有初稿，但仍有开放问题 |
| `specified` | `spec.md` 已定义主要 Scenario 和契约 |
| `ready` | `plan.md` 已完成，切片可执行 |
| `implementing` | 至少一个 Slice 正在实现 |
| `escaping` | 有实现反馈需要回到 Spec 或 Plan |
| `verifying` | 正在做最终一致性校验 |
| `archived` | 已同步主 Spec 并归档 |

### 8.2 Slice 状态机

```text
pending
  |
  v
claimed
  |
  v
red
  |
  v
green
  |
  v
refactor
  |
  v
reviewing
  |
  v
done

blocked -> escaped -> pending
```

状态说明：

| 状态 | 含义 |
|---|---|
| `pending` | 尚未开始 |
| `claimed` | 当前 agent 或子 agent 已认领 |
| `red` | 已写失败测试，正在确认失败原因正确 |
| `green` | 最小实现已通过相关测试 |
| `refactor` | 全绿下做内部结构改进 |
| `reviewing` | 做 Spec compliance 和 code quality review |
| `done` | 切片完成且证据已记录 |
| `blocked` | 缺上下文、依赖、环境或实现障碍 |
| `escaped` | 已触发 Escape，等待 Spec/Plan 修正 |

### 8.3 Escape 状态机

```text
opened
  |
  v
spec_or_plan_updated
  |
  v
test_strategy_updated
  |
  v
closed
```

### 8.4 Review 状态机

```text
requested -> issues_found -> fixed -> requested
requested -> approved
requested -> accepted_with_risk
```

`accepted_with_risk` 只能用于 Warning 或 Suggestion。Critical 必须修复，或由用户显式确认风险。

## 9. 并行实现模型

### 9.1 可并行条件

两个 Slice 同时满足以下条件才可并行：

- 无依赖关系。
- 写入范围不重叠。
- 不同时修改同一外部契约。
- 不共享数据库迁移、全局配置、公共类型等高冲突资源。
- 测试运行不会互相污染。
- 每个 Slice 有清晰 Scenario 和验收证据。

### 9.2 并行委派输入

对子 agent 的最小上下文：

```text
Change:
Slice:
Scenario IDs:
External contract:
Write scope:
Do-not-touch scope:
Test strategy:
TDD requirement:
Expected evidence:
Escalation rules:
```

### 9.3 并行后集成

controller 必须执行：

- 检查每个子 agent 实际修改范围。
- 跑相关测试。
- 跑跨 Slice 集成测试。
- 处理冲突或重复实现。
- 更新 `state.json`。

不能只相信子 agent 的成功报告。

## 10. 验证与证据

### 10.1 Slice 完成证据

每个 Slice 至少记录：

- 失败测试曾经失败的证据，或说明为何无法 Red-first。
- 通过的相关测试命令。
- 对应 Scenario ID。
- 代码 review 结果。
- 未验证项和原因。

### 10.2 Change 完成证据

`/flow:close` 前至少检查：

| 编号 | 检查 |
|---|---|
| V1 | `brief.md` 的目标和范围仍然成立 |
| V2 | `spec.md` 的主要 Scenario 有测试或明确验证证据 |
| V3 | 外部契约和实现一致 |
| V4 | `plan.md` 的 Slice 均完成或明确移出范围 |
| V5 | Escape 全部关闭 |
| V6 | 关键测试通过 |
| V7 | review 的 Critical 已处理 |
| V8 | 主 Spec 已同步或明确跳过并记录风险 |

## 11. 风险边界

### 11.1 不适合完整流程的场景

以下场景应降级为轻量 TDD、探索或直接修复：

- 一次性脚本、临时工具。
- 纯原型、Demo、短期验证。
- 需求每天剧烈变化，尚无稳定业务语言。
- 只改内部实现且不改变外部行为的小重构。
- P0 热修复，需要先止血。

### 11.2 必须走完整流程的场景

以下场景建议完整执行：

- 新功能或重要业务行为变化。
- 外部 API、事件、数据模型或错误语义变化。
- 权限、安全、合规、审计相关变化。
- 多模块、多团队、多 repo 协作。
- 高回归风险路径。
- AI 并行实现密集的变更。

### 11.3 文档膨胀风险

控制规则：

- 默认只创建 `brief.md`、`spec.md`、`plan.md`、`state.json`。
- ADR 只记录难逆转、非显然、有真实权衡的决策。
- `CONTEXT.md` 只放领域语言，不放需求和实现计划。
- `spec.md` 不写内部实现细节。
- `plan.md` 不写长期会过期的逐行代码脚本。

### 11.4 TDD 误用风险

禁止：

- 一次性批量写完所有测试再批量实现。
- 为私有实现细节写脆弱测试。
- 测试通过后反向修改 Spec 以适配实现。
- Red 状态下重构。
- 跳过失败测试证据后声称完成。

允许：

- 对纯配置、生成代码、探索原型降级，但必须记录原因。
- 对无合适测试缝隙的 legacy code，先建立更高层验证或记录架构缺陷。

### 11.5 Spec 误用风险

禁止：

- 把 Spec 写成内部类图或函数清单。
- 把未验证的探索想法写成稳定契约。
- 为了追踪方便强制每个 unit test 绑定 Scenario。
- 在实现中发现契约问题却不触发 Escape。

### 11.6 并行实现风险

禁止并行：

- 多个 Slice 修改同一文件或同一公共类型。
- 多个 Slice 同时修改同一 Requirement。
- 数据迁移、鉴权、安全策略等高耦合变更。
- 测试环境共享且会互相污染。

需要串行：

- 先建立基础接口或数据模型，再实现依赖行为。
- 先修正 Spec 或 Plan，再继续实现。
- 先完成 contract test，再实现外部适配。

### 11.7 Escape 过多风险

治理规则：

- 单个 change Escape 超过 3 次，应重新审查 `spec.md` 和 `plan.md`。
- 超过一半 Slice 触发 Escape，应回到 `/flow:clarify`。
- 同类 Escape 反复出现，说明团队领域语言、Scenario 粒度或切片方式有系统问题。

## 12. 变更类型处理策略

| 变更类型 | 推荐流程 |
|---|---|
| 新功能 | `clarify -> spec -> plan -> apply -> close` |
| 外部契约变化 | 完整流程，并优先添加 contract / integration tests |
| Bug 修复且改变外部行为 | `clarify` 可轻量，必须补 `spec` 和 TDD |
| Bug 修复且只是不符合现有 Spec | 可从 `plan/apply` 开始，直接 TDD 修复 |
| 内部重构 | 通常不需要 `spec`，只执行 refactor with tests |
| 性能优化 | 若有外部性能承诺，写入 `spec`；否则轻量 plan + benchmark |
| 热修复 | 先补必要测试和代码，事后补 `brief/spec/plan` 的最小记录 |
| 原型探索 | 可只用 `/flow:clarify`，不要归档进主 Spec |

## 13. 与 issue tracker 的关系

Issue tracker 是协作分发工具，不是核心事实来源。

推荐关系：

- `brief/spec/plan` 成熟后，可把 Slice 导出为 issue。
- Issue body 引用 change 和 Slice ID。
- Issue 完成后，证据回写到 `state.json` 或 `plan.md`。
- 主事实仍在 change folder 和主 Spec。

这样保留 Matt-skills 的 issue 协作能力，但避免 issue tracker 替代 Spec。

## 14. 最小可落地版本

### L0：人工可启动

- 手写 `brief.md`、`spec.md`、`plan.md`。
- `state.json` 可简化，仅记录 Slice 状态。
- `/flow:*` 先作为 agent skills，不必实现 CLI。
- PR 模板包含 close checklist。
- 对一次性小改动允许轻量路径：`/flow:clarify` 后直接 `/flow:apply` 生成单 Slice，但不得直接归档进主 Spec；若要归档，必须补齐 `spec.md` 和 `plan.md`。

### L1：轻量治理

- Scenario ID 稳定。
- 关键测试可引用 Scenario ID。
- `state.json` 记录 Escape、Review、Evidence。
- close 前检查未关闭 Escape 和未完成 Slice。

### L2：工程化治理

- 自动解析 `plan.md` Slice 和 write scope。
- 自动判断并行冲突。
- CI 检查 Scenario 引用悬空。
- 外部契约变化要求测试证据。

### L3：高治理场景

- Spec/Test trace graph。
- Scenario hash 或版本检查。
- Escape 频率统计。
- LLM 辅助语义一致性检查。

## 15. 最终原则

```text
clarify 消除需求歧义。
spec 稳定外部契约。
scenario 连接 Spec 与 Test。
plan 拆出行为切片和并行边界。
apply 在切片内执行 TDD。
escape 把实现反馈回流到 Spec。
review 提前发现偏离。
close 证明 Spec / Test / Code 语义一致并归档。
```

这套流程的关键不是命令数量，也不是文档数量，而是三件事：

- Spec 是否表达真实业务意图和外部承诺。
- Test 是否提供可信的可执行证据。
- Code 是否以可维护结构实现这些行为。
