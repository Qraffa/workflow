# 统一工作流方案：SliceSpec

> 基于 final-workflow.md 的 SDD+TDD 融合思想，提炼并整合 OpenSpec / matt-skills / superpowers 三个工作流的核心优势，落地为一套精简、可工程化的命令管线。

---

## 0. 速览

| 维度 | 设计选择 |
|---|---|
| 核心理念 | SDD 定义契约 + TDD 驱动实现 + Subagent 并行执行 |
| 命令总数 | **6 个**（`/clarify`、`/spec`、`/slice`、`/implement`、`/escape`、`/verify`） |
| 主要产物 | **4 份人读文档 + 1 份机器状态** (`brief.md`、`spec.md`、`slices.md`、`escapes.log`、`state.json`) |
| 持久语义 | `CONTEXT.md`（领域语言，可选）+ `specs/<capability>/spec.md`（archive 后合并目的地） |
| 并行模型 | 切片级（subagent per slice），HITL/AFK 分类 + write_scope 静态隔离 |
| Verify 单位 | Scenario（Given/When/Then）作为 Spec↔Test 桥接，ID 在 archive 后不可变 |
| 最小启动成本 | L0 模式下 `/clarify` + `/implement` 即可启动，brief 与 state 在 L1+ 强制 |

**比较参考（命令数与文件数）：**

| 工作流 | 主路径命令数 | 单次 change 人读文档 | 机器状态 |
|---|---|---|---|
| OpenSpec | 11 | 4 (proposal + design + tasks + spec) | 无（依赖文件存在性扫描） |
| matt-skills | 14（5 主线 + 9 辅助） | N 个 issue（issue tracker 即状态） | 无 |
| superpowers | 14 | 2 (design + plan) | TodoWrite（会话级） |
| 替代方案（integrated-sdd-tdd） | 7 | 3 (brief + spec + plan) | state.json |
| **SliceSpec (本方案)** | **6** | **4 (brief + spec + slices + escapes)** | state.json |

---

## 1. 设计原则

1. **精简优先**：命令与产物数量在能覆盖 final-workflow.md 全部 17 章的前提下最少化。每多一个命令、多一个文件，必须证明它解决了一类无法被现有产物覆盖的问题。
2. **契约与实现分离**：spec 关心"对外承诺"，slice 关心"对内拆分"，二者通过 Scenario ID 互相引用，但任一方变动不会强制另一方同步重写。
3. **Soft 依赖默认**（继承自 matt-skills）：所有命令在缺失上游产物时给出"自动 fallback / 提示用户先跑哪步"的行为，而不是硬报错。仅 `/verify` 在严格模式下强制依赖完整。
4. **Subagent 化并行**（继承自 superpowers）：主会话只做调度，切片实现交给 fresh subagent，避免 context 污染。HITL 切片显式回到主会话。
5. **Escape Hatch 是一等公民**：任何阶段都可以触发 `/escape`，记录为 append-only 日志，verify 阶段必须确认闭环。
6. **Skill 是场景化的，不接受 CLI 风格 flag**：用自然语言意图调用（如"继续下一个 slice"、"严格校验"），不用 `--strict`、`--bulk` 这类参数。Skill 内部依据 `state.json` 与意图判断行为模式。
7. **并行隔离靠静态声明，不靠子 agent 自觉**：每个 slice 必须自报 `write_scope`（白名单）与 `do_not_touch`（黑名单）。controller 在子 agent 返回后必须 `git diff` 校验越界，越界即视为失败而非概念性 review issue。
8. **Scenario ID 是稳定契约的一部分**：ID 在 archive 后不可变；新增 Scenario 走 `## ADDED`，废弃走 `## REMOVED` 并保留 reason，绝不复用已 archive 的 ID 命名。这是 Spec↔Test 长期可追溯的基础。

---

## 2. 命令清单

### 2.1 `/clarify` —— 需求澄清（Explore + Brainstorm + Grill）

**取代**：matt-skills 的 `/grill-me` + `/grill-with-docs` + superpowers 的 `brainstorming` + OpenSpec 的 `/opsx:explore`。

**何时调用**：
- 用户对"做什么/为什么做/边界在哪"还没说清。
- spec 已经存在但用户想挑战它（重新澄清边界）。
- `/escape` 治理阈值超限触发"退回澄清"建议时。

**输入**：用户的自然语言描述、相关代码路径（可选）、issue 链接（可选）。

**行为**：
1. 如果存在 `CONTEXT.md`，先读它，并在对话中用其中的术语；遇到术语冲突时主动澄清。
2. 一次一个问题（继承 matt-skills 的 Socratic 模式），每个问题附 Claude 的推荐答案。
3. 鼓励用户在每个分支决策上回答 "确认 / 修正 / 让 Claude 决定"。
4. 当达到"可以写 spec 草稿"的阈值（用户显式同意 OR 5 个连续无修正），自动建议 `/spec`。

**产物**：
- `changes/<change-id>/brief.md`（L1+ 强制；L0 模式可省略）—— 含 Problem / Goal / Scope / Non-Goals / Domain Language / Constraints / Open Questions / Decision Log 8 个章节
- `changes/<change-id>/state.json` 初始化（含 change_id、status=`draft`、updated_at）

**brief.md 模板**：

```markdown
# Brief: <change-id>

## Problem
<1-3 句话，业务视角>

## Goal
<本次 change 要达成的可观察目标>

## Scope
<明确包含的能力/流程/接口>

## Non-Goals
<明确不做什么，防止隐性扩张>

## Domain Language
<本 change 涉及的关键术语；与 CONTEXT.md 冲突时此处为局部约定>

## Constraints
<性能阈值、合规要求、迁移约束、外部依赖>

## Open Questions
<未决议的歧义；进入 /spec 前应清零或显式标记为"实现期再定">

## Decision Log
<- YYYY-MM-DD: 决定 X，因为 Y，否决了 Z>
```

**与 final-workflow.md 的映射**：第 4 节 "Explore / Clarify"。

**Soft 依赖**：零配置可用。在 L0 模式下 brief.md 可省略，结论留在会话上下文中；进入 `/spec` 时若发现没有 brief.md，由 `/spec` 在生成 spec.md 时反向回填一个最小 brief.md（仅 Problem/Goal/Scope/Non-Goals 4 节）。

---

### 2.2 `/spec` —— 定义契约（SDD 核心）

**取代**：OpenSpec 的 `/opsx:propose` + `/opsx:new` + `/opsx:continue`（合并）。

**何时调用**：`/clarify` 后；或者用户明确知道要做什么，跳过澄清直接写 spec。

**输入**：来自 `/clarify` 的对话上下文 + 用户对边界的最终确认。

**行为**：
1. 创建 `changes/<change-id>/spec.md`（kebab-case id）。
2. 自动判断 brownfield vs greenfield：
   - 查 `specs/` 目录是否存在相关 capability。
   - 存在 → 用 OpenSpec 的 **delta 语法**（`## ADDED/MODIFIED/REMOVED/RENAMED Requirements`）。
   - 不存在 → 生成全新 capability，标记为 `## ADDED Requirements`。
3. 强制 final-workflow.md 第 5 节的 Scenario 格式：
   ```
   ### Requirement: <name>
   <The system SHALL …>

   #### Scenario: <name>
   - **GIVEN** <preconditions>
   - **WHEN** <trigger>
   - **THEN** <observable outcome>
   ```
4. 为每个 Scenario 分配稳定 ID（`<capability>.<requirement-slug>.<seq>`），便于后续测试引用。**ID 稳定性规则**：
   - archive 后 ID 不可变；后续 change 即使重写该 Scenario 必须保留原 ID。
   - 废弃 Scenario 走 `## REMOVED Requirements` 并标 `**Reason**` 与 `**Migration**`，**绝不复用其 ID 命名给新 Scenario**。
   - 跨 capability 引用 Scenario ID 时使用 fully qualified 形式（`auth.login.001`），不允许局部短名。
   - 测试中引用 ID 至少满足以下任一形式：注释 `// @scenario: auth.login.001`、测试名前缀 `test_scenario_auth_login_001_*`、或 test docstring 首行 `Scenario: auth.login.001`。`/verify` 三选一接受。
5. spec.md 必含 4 个章节：**Why / Non-Goals / Requirements / Constraints**。**显式禁止**：实现细节、私有类、目录结构、库选择（除非这些本身是外部约束）。

**产物**：`changes/<change-id>/spec.md`

**与 final-workflow.md 的映射**：第 2.1 / 3 / 5 节。

**Soft 依赖**：建议但不强制存在 `CONTEXT.md`；缺失时使用项目根的 README 提取术语。

**关键设计取舍**：
- 合并了 OpenSpec 的 `proposal.md` 与 `design.md` 进单个 `spec.md`：proposal 的"Why"成为开头章节，design 的"Decisions/Risks/Migration"作为 spec 末尾的可选附录。这砍掉一个文件但**保留了对外契约的完整表达**。
- 不强制 `design.md` 独立成文：only 当涉及跨服务、外部依赖、安全/性能/迁移复杂度时，rationale 写入 spec.md 的 **Constraints** 章节即可。

---

### 2.3 `/slice` —— 行为切片与任务拆分

**取代**：matt-skills 的 `/to-issues` + OpenSpec 的 tasks.md 生成 + superpowers 的 `writing-plans`。

**何时调用**：spec.md 完成后；或者用户在 spec 不完整时也可以先切片探索可行性（此时切片将标注 `[draft]` 提示）。

**输入**：`changes/<change-id>/spec.md`。

**行为**：
1. 读 spec.md 中所有 Requirements，按 final-workflow.md 第 6 节优先级排序（高业务价值 → 高风险 → 外部契约 → 易回归 → 基础依赖）。
2. 提出 tracer-bullet 垂直切片清单，每个 slice 包含：
   - **id**：`<change-id>-s<NN>`
   - **覆盖的 Scenario IDs**：必须显式列出
   - **type**：HITL（需人决策）/ AFK（可由 subagent 独立完成）
   - **blocked_by**：依赖关系
   - **write_scope**：白名单 glob 列表（如 `src/auth/**`、`tests/auth/**`），子 agent 仅允许在该范围内创建/修改文件
   - **do_not_touch**：黑名单 glob（默认包含 `specs/**`、`changes/**`、`.github/**`、`CONTEXT.md` 等共享资源；可追加）
   - **test_strategy**：用 final-workflow.md 第 7 节的分层（acceptance/integration/contract/unit）
   - **estimated_cycles**：预计 TDD 循环数（粗估，<10 提示拆分）
3. 交互式让用户确认/调整。
4. 生成 `changes/<change-id>/slices.md`。

**产物**：`changes/<change-id>/slices.md`（结构化 markdown，含 checkboxes 以便 `/implement` 追踪进度）。

**slices.md 模板**：

```markdown
# Slices for <change-id>

## Dependency Graph
```
s01 ──┬──> s03 ──> s05
      └──> s04
s02 ─────────────────> s05
```

## Slice s01: <name>
- **type**: AFK
- **covers**: auth.login.001, auth.login.002
- **blocked_by**: none
- **write_scope**:
  - src/auth/**
  - tests/auth/**
- **do_not_touch**:
  - specs/**
  - src/billing/**
- **test_strategy**:
  - acceptance: 用户成功登录路径
  - unit: 密码错误次数计数
- **estimated_cycles**: 3
- **status**: pending  <!-- 见 §3.2 切片状态机 -->

## Slice s02: ...
```

**与 final-workflow.md 的映射**：第 6 / 7 节。

**关键设计取舍**：
- 不分离 `tasks.md` 与 `plan.md`：superpowers 的 plan 含"每步完整代码块"对 LLM 实现派发非常重要，但对人类 review 太冗长；OpenSpec 的 tasks.md 又过于扁平。**slices.md 只到 slice 粒度（行为），完整步骤代码下沉到 `/implement` 内部由 subagent 自己写**。这避免在切片阶段 over-engineer。
- HITL/AFK 标注（matt-skills 优秀实践）：直接决定 `/implement` 是分派 subagent 还是回到主会话。

---

### 2.4 `/implement` —— 切片实现（TDD + Subagent）

**取代**：superpowers 的 `subagent-driven-development` + `executing-plans` + matt-skills 的 `/tdd` + OpenSpec 的 `/opsx:apply`。

**何时调用**：slices.md 存在且至少一个 slice 是 `pending`。

**输入**：`changes/<change-id>/slices.md` + slice id（可选；默认按 dep graph 取下一个 unblocked AFK slice）。

**行为**（双模式）：

#### 模式 A：AFK 切片（自动派发）

1. 主会话读 `slices.md`，取下一个 `unblocked + AFK + pending` 切片。
2. 提取该 slice 引用的 Scenario IDs，从 spec.md 复制对应 Requirement 全文。
3. 派发 **implementer subagent**（继承 superpowers 的 `implementer-prompt.md` 模板），prompt 包含：
   - slice 全文 + 关联 Scenario（Given/When/Then）
   - **强制 TDD 循环**（matt-skills 的 `tdd` 协议 + superpowers 的"NO CODE WITHOUT FAILING TEST FIRST"铁律）：
     - Red：写一个针对当前 Scenario 的失败测试
     - Green：写最少代码让测试通过
     - Refactor：在全绿下重构
     - **重复直到 slice 完成**
   - 测试中必须用注释引用 Scenario ID（如 `// @scenario: auth.login.001`）
   - 报告格式：`DONE / DONE_WITH_CONCERNS / BLOCKED / NEEDS_CONTEXT`（继承 superpowers）
4. 派发 **spec-compliance reviewer subagent**（继承 superpowers 的 `spec-reviewer-prompt.md`），独立读 spec.md 与实现代码，验证：
   - 是否覆盖所有声明的 Scenario？
   - 是否引入了 spec 未声明的外部行为？
   - 测试是否引用了 Scenario ID？
5. 如果 reviewer 报 issue → implementer 修复 → reviewer 复查（最多 3 轮，超过则升级为 BLOCKED）。
6. 派发 **code-quality reviewer subagent**：
   - 测试是否通过公共接口验证（非 mock 内部）？
   - 是否过度抽象 / 速决性代码？
   - 是否在 RED 时重构？（违反检测）
7. **Controller 边界校验**（不可省略，先于 reviewer 派发）：主会话执行 `git diff --name-only` 取子 agent 实际修改的文件列表，逐一比对 slice 的 `write_scope` 白名单与 `do_not_touch` 黑名单。
   - 任一文件未匹配白名单 → 标记 slice 为 `blocked(scope-violation)`，不进入 reviewer，要求子 agent 撤回越界修改后重新 dispatch。
   - 任一文件命中黑名单 → 同上。
   - **这是结构性失败，不交给 reviewer 用自然语言判断。**子 agent 自我汇报的"我没改 X"不可信。
8. 通过 reviewer → 更新两处状态：
   - `slices.md` 的 status 推进到 `done`。
   - `state.json` 的对应 slice 节点写入 `evidence`（包含 commit SHA 列表、测试命令与结果、reviewer 结论），并写入 `updated_at` 时间戳。
9. 如果还有 unblocked AFK slice → 继续派发下一个；否则提示 `/verify`。

#### 模式 B：HITL 切片（主会话执行）

1. 主会话直接执行 TDD 循环，每个 cycle 与用户对齐。
2. 一旦 HITL 切片完成，回到模式 A 继续后续 AFK 切片。

#### HITL → AFK 降级

HITL 切片在以下条件全部满足时，可在执行中段降级为 AFK 后续派发给 subagent：

1. 用户已对该切片的关键决策点（架构选择、UX 权衡、外部依赖确认等）给出明确答复，且答复内容能在 prompt 中复述。
2. 切片剩余工作均为 mechanical 实现（具体测试编写、代码改动），不再有需要人裁决的分支。
3. `write_scope` 仍然满足并行隔离条件。

**判定权**在 controller（主会话），不在 subagent。降级时必须：
- 在 `state.json.slices[sid]` 新增 `originally_hitl: true` 字段（审计用）。
- 在 dispatch 给 subagent 的 prompt 中显式列出"用户已确认的决策"清单，作为不可变前置条件。
- 降级后子 agent 若发现新的 HITL 决策点，必须以 `BLOCKED(needs-hitl-decision)` 退出而非自行决定。

反向不允许：AFK 切片在实现中发现需要人裁决时，走 `/escape` 通道而非降级为 HITL。

#### 并行性

**判定主体**：始终是 controller（主会话），不交给 subagent 自决。

**判定算法**（按顺序短路，任一不通过即降级为串行）：

1. **依赖检查**：候选切片集合 `S` 中两两之间 `blocked_by` 无路径冲突 → 通过。
2. **write_scope 集合不相交**：对 `S` 中每两个切片 `(a, b)`，`write_scope(a) ∩ write_scope(b) = ∅`（glob 展开后比对）→ 通过。仅文件级，目录级冲突也算冲突。
3. **do_not_touch 黑名单不相交**：`do_not_touch(a) ∩ write_scope(b) = ∅`，反之亦然 → 通过。
4. **共享高冲突资源检测**：扫描 `S` 中各切片 write_scope 是否触及以下默认高冲突路径，触及即不可并行：
   - 数据库 migration 目录（`**/migrations/**`、`db/migrate/**`）
   - 全局配置（`**/config/**`、`*.toml`、`*.yaml` 在根）
   - 公共类型/schema 文件（`**/types/**`、`**/*.proto`、`**/*.graphql`）
   - 用户可在 `state.json.parallel_guards` 追加项目特定路径。
5. **测试隔离物理保证**：每个并行切片必须分配独立 git worktree（继承 superpowers 的 `using-git-worktrees`），并在 worktree 内运行各自测试。**这是物理隔离，不依赖"测试不互相污染"的人为承诺**。共享 DB/外部服务的项目需在 slices.md 标注 `requires_shared_resource: <name>`，命中即不可并行。

**判定不出来时默认串行**。controller 不做"乐观并行"——风险大于收益。

**合并策略**：每个并行切片完成后单独 commit；所有切片 done 后，controller 把各 worktree 的 commit cherry-pick 回主 worktree，冲突由 controller 处理（不交给 subagent）。`/verify` 在合并后的代码上运行。

**产物**：
- 测试 + 实现代码（按项目结构）
- git commits（每个 TDD 循环至少一个 commit）
- `changes/<change-id>/evidence/<slice-id>/*.md`（implementer/spec-reviewer/quality-reviewer 各一份长文报告）
- 更新 `slices.md` 的 status checkbox 与 `state.json` 的 evidence 索引

**与 final-workflow.md 的映射**：第 4 (切片内执行 TDD) / 8 (切片内 TDD 执行规则) / 9 (Refactor 边界规则) 节。

**Soft 依赖**：必须有 `slices.md`；若用户跳过 `/slice` 直接 `/implement`，自动 fallback 为生成一个临时单切片并提示用户补 spec。

**关键设计取舍**：
- TDD 不作为单独命令，**而是 `/implement` 的强制内嵌行为**。理由：tdd 是实现纪律，不是阶段；让它单独存在容易被跳过。superpowers 的做法（plan 模板硬绑定 RED-GREEN-REFACTOR）证明这是有效的。
- 默认双 reviewer（spec → quality）：增加约 2x 成本但显著提升一次通过率（参考 superpowers RELEASE-NOTES 的实测数据）。用户可用自然语言降级（"这个 change 跳过 quality review" / "只做 spec compliance"），skill 据此调整；不暴露 flag。
- 不强制并行：用户/项目复杂度低时，串行 AFK 也是合法路径。并行是性能优化，不是正确性要求。

---

### 2.5 `/escape` —— Escape Hatch

**取代**：final-workflow.md 第 10 节专门设计的机制，**三个原工作流都没有显式对应物**。

**何时调用**：`/implement` 过程中触发 final-workflow.md 第 10 节列举的任一场景：
- Scenario 语义错误、歧义或矛盾
- 测试发现 spec 未覆盖的外部可观察边界
- 重构中浮现出更合适的外部接口或契约
- Bug 修复会改变外部行为或验收标准
- 实现成本明显超出预期

**输入**：当前所在 slice id + escape 标签（`spec-error` / `scenario-missing` / `better-interface` / `scope-overflow`） + 自由文本描述。

**行为**：
1. **暂停**当前 implementer subagent（让它以 BLOCKED 退出并保留进度）。
2. 在 `changes/<change-id>/escapes.log` append 一条记录（**永不删除**，archive 时保留）：
   ```
   ---
   ts: 2026-05-19T10:23:00Z
   slice: cc-analy-s03
   tag: scenario-missing
   description: |
     登录失败 5 次后应该锁账户，但 spec.md 的 auth.login Requirements
     完全没覆盖这个场景。
   resolution_pending: yes
   ---
   ```
3. 进入 **mini-spec-update** 模式（轻量 spec 修订）：
   - 仅允许修改与该 escape 直接相关的 Requirement / Scenario。
   - 修改后回写 spec.md，并把 escapes.log 中该条目的 `resolution_pending` 改为 `no`。
4. 自动更新 slices.md：标记当前 slice 为 `escaped`，可能新增 follow-up slice。
5. 提示用户：恢复原 slice（`/implement <slice-id>`）或继续后续 slice。

**产物**：
- `changes/<change-id>/escapes.log`（append-only）
- spec.md 的增量修订（git diff 可追溯）

**治理（继承 final-workflow.md 第 10 节）**：
- 单 change > 3 次 escape：`/verify` 触发 review 警告。
- 单 change 超过一半 slice 触发 escape：`/verify` 阻塞，建议退回 `/clarify`。

**与 final-workflow.md 的映射**：第 10 节（直接实现）。

**关键设计取舍**：
- 让 escape 成为**独立命令**而不是 spec.md 的隐式编辑：保留可审计的轨迹。否则团队很难发现"这个 change 已经偏离原 scope 太远"的信号。
- 命名为 `/escape`（不是 `/replan`）强调它是**逃生通道，不应高频使用**。

---

### 2.6 `/verify` —— 一致性校验与归档

**取代**：OpenSpec 的 `/opsx:verify` + `/opsx:sync` + `/opsx:archive` + superpowers 的 `finishing-a-development-branch`（合并为单一终结命令）。

**何时调用**：slices.md 中所有 slice status 为 `done` 或 `escaped`（已闭环）。

**行为**：
1. **结构化校验**（继承 OpenSpec 的 `validate`）：
   - spec.md 中每个 Requirement 至少有一个 Scenario。
   - 每个 Scenario 是 4 个 `#`（不是 3 个），用 `**GIVEN**/**WHEN**/**THEN**` 格式。
   - delta 标记（ADDED/MODIFIED/REMOVED/RENAMED）无跨节冲突。
2. **语义一致性校验**（来自 final-workflow.md 第 12 节 V1-V8）：
   - V1：spec.md 是否仍表达真实业务意图（用 LLM 重新扫描，flag 模糊语言）
   - V2：每个 active Scenario 是否有对应测试引用其 ID（grep `@scenario:` 或注释）
   - V3：外部接口、错误语义、数据模型、权限、安全、性能承诺与 spec.md 一致
   - V4：是否有 acceptance/integration/contract test 覆盖主要验收路径
   - V5：unit test 是否覆盖关键规则、边界
   - V6：`escapes.log` 中所有 `resolution_pending: yes` 均已闭环
   - V7：TDD 中发现的新业务语义已回填 spec.md（检查 git history vs spec diff）
   - V8：spec.md 未夹带内部实现细节
3. **Escape 治理检查**（来自 final-workflow.md 第 10 节阈值）：
   - escapes 数 > 3：警告
   - escapes 数 > slices/2：阻塞
4. **测试套件运行**：本地 + CI（参考 final-workflow.md 第 8 节本地验证策略）。
5. **Sync + Archive**：
   - Sync：把 `changes/<change-id>/spec.md` 中的 delta 合并到 `specs/<capability>/spec.md`（继承 OpenSpec 的 requirement 粒度 merge 算法）。
   - Archive：把 `changes/<change-id>/` 移到 `changes/archive/YYYY-MM-DD-<change-id>/`。
   - Commit + 可选地 `gh pr create`。

**产物**：
- 验证报告（stdout，可选写入 `changes/<change-id>/verify-report.md`）
- 主 `specs/` 仓库的更新
- archive 后的 change 目录

**场景化调用**（自然语言，非 CLI flag）：
- "PR 前自检一下" → 仅运行 V1-V8 与测试套件，不执行 sync/archive。
- "严格校验" → Warning 视为阻塞（行为定义见下表），Suggestion 仍仅提示。
- "归档所有完成的 change" → 扫描 `changes/` 下 status=`verifying` 的所有 change，批量执行 sync+archive。
- "归档当前 change" → 默认行为。

**严格度对照**（避免"strict"措辞含糊）：

| 严重度 | 默认模式 | 严格模式 |
|---|---|---|
| Critical | 阻塞 | 阻塞 |
| Warning | 提示 + 可 `accepted_with_risk` | 阻塞（不能 accepted_with_risk）|
| Suggestion | 提示 | 提示 |

`/verify` 输出报告同时给出退出码（§"机器可读输出"），CI 据此决定合并。

**机器可读输出（CI 集成接口）**：
- 退出码：`0` 通过，`1` 有 Critical，`2` 有 Warning 且严格模式开启，`3` 结构性错误（spec 解析失败等）。
- 报告路径：`changes/<change-id>/verify-report.json`（结构化）+ `changes/<change-id>/verify-report.md`（人读）。
- CI 可直接 `<flow:verify cmd> && cat changes/*/verify-report.json | jq ...`。

**与 final-workflow.md 的映射**：第 12 节（直接实现 V1-V8）。

**关键设计取舍**：
- **不单独提供 `/sync`**：超过 99% 场景 sync 与 archive 同时发生；想要"sync 早，archive 晚"的边界场景太罕见，让用户手动 git 操作即可。
- **不单独提供 `/diagnose`**：实现期间发现 bug 应通过 `/escape` 路径处理（如果改变外部行为）或在 implementer subagent 内修复（如果是纯内部 bug）。matt-skills 的 `diagnose` 强大但与本工作流的核心任务流相关性低，**作为可选独立 skill 保留**，不进核心命令集。

---

## 3. 状态机与产物流

### 3.1 Change 状态机（5 个状态，记录在 state.json）

```
draft ──> specified ──> implementing ──> verifying ──> archived
                            │
                            └─ escape 是子状态，不构成独立 change 状态
```

| 状态 | 含义 | 进入条件 | 主要命令 |
|---|---|---|---|
| `draft` | 已有 brief.md，目标范围未完全稳定 | `/clarify` 完成首轮 | `/clarify` 迭代 |
| `specified` | spec.md 存在且 Scenario 有稳定 ID | `/spec` 完成 | `/slice` |
| `implementing` | slices.md 存在且至少一个 slice in_progress | `/slice` 完成或首次 `/implement` | `/implement` / `/escape` |
| `verifying` | 所有 slice 状态为 done 或 escaped(closed) | `/implement` 完成最后一个 slice | `/verify` |
| `archived` | sync + archive 完成 | `/verify` 通过 | (不可逆) |

### 3.2 Slice 状态机（细化到 TDD 节拍）

```
pending ──> in_progress ──┬──> done
                          │     ▲
                          │     │ (reviewer 通过 + controller diff 校验通过)
                          │     │
                          │     ├── reviewing      ← review 阶段
                          │     ├── refactor       ← 全绿下重构
                          │     ├── green          ← 最小实现通过测试
                          │     └── red            ← 失败测试就位
                          │
                          ├──> blocked (上下文缺失 / scope-violation / 依赖未就绪)
                          │
                          └──> escaped (open) ──> escaped (closed) ──> pending
```

| 状态 | 含义 | 持有方 |
|---|---|---|
| `pending` | 未开始 | — |
| `in_progress.red` | 失败测试已落地并验证按预期失败 | subagent |
| `in_progress.green` | 最小实现通过该测试 | subagent |
| `in_progress.refactor` | 全绿下重构 | subagent |
| `in_progress.reviewing` | 双 reviewer 派发中 | controller |
| `done` | reviewer 通过 + diff 校验通过 + evidence 落 state.json | — |
| `blocked` | 子原因：`needs-context` / `scope-violation` / `dep-missing` / `too-complex` | controller |
| `escaped(open)` | 已触发 `/escape`，等待 spec/slices 修订 | — |
| `escaped(closed)` | spec/slices 已修订，slice 可重启或被新 slice 替代 | — |

**为什么细分 in_progress 而不平铺**：所有 `in_progress.*` 在调度视角下是同一类（"该 slice 占用一个 subagent"），细分仅用于 controller 在 subagent 报告中提取进度。在 state.json 中用 `state.json.slices[sid].phase` 字段记录细分；slices.md 的人读 checkbox 只显示 `pending/in_progress/done/blocked/escaped` 五大类。

### 3.3 state.json 结构

`state.json` 是机器状态文件，不是人读入口（人读看 brief/spec/slices）。schema：

```json
{
  "change_id": "add-auth-login",
  "status": "implementing",
  "created_at": "2026-05-19T10:00:00Z",
  "updated_at": "2026-05-19T15:30:00Z",
  "current_slice": "add-auth-login-s02",
  "slices": {
    "add-auth-login-s01": {
      "status": "done",
      "phase": null,
      "owner": null,
      "scenarios": ["auth.login.001"],
      "write_scope": ["src/auth/**", "tests/auth/**"],
      "do_not_touch": ["specs/**"],
      "evidence": {
        "commits": ["a1b2c3d", "e4f5g6h"],
        "tests_run": [
          {"cmd": "pytest tests/auth/test_login.py -v", "result": "pass", "ts": "..."}
        ],
        "spec_review": "approved",
        "quality_review": "approved",
        "controller_diff_check": "passed"
      }
    },
    "add-auth-login-s02": {
      "status": "in_progress",
      "phase": "red",
      "owner": "subagent:implementer-7f3a",
      "scenarios": ["auth.login.002"],
      "write_scope": ["src/auth/**"],
      "evidence": {}
    }
  },
  "escapes": [
    {
      "id": "e01",
      "ts": "...",
      "slice": "add-auth-login-s01",
      "tag": "scenario-missing",
      "description": "...",
      "resolved": true,
      "resolved_at": "..."
    }
  ],
  "reviews": [
    {
      "object": "slice:add-auth-login-s01",
      "verdict": "approved",
      "severity_breakdown": {"critical": 0, "warning": 1, "info": 2},
      "warnings_accepted_with_risk": ["magic-number-100 in auth/login.py:42"]
    }
  ]
}
```

**关键设计**：
- 人手不应直接编辑 state.json；所有更新由 `/clarify`、`/spec`、`/slice`、`/implement`、`/escape`、`/verify` 各自维护对应字段。
- `evidence.controller_diff_check` 字段记录 §2.4 第 7 步的边界校验结果，是并行隔离的审计证据。
- `reviews[].warnings_accepted_with_risk` 字段实现"显式接受风险"机制（吸收自 integrated-sdd-tdd 的 `accepted_with_risk` review 状态），仅 Warning 级可接受，Critical 必须修复。

**state.json 中 evidence vs `evidence/` 目录的职责边界**（避免重复存储）：

| 内容 | 存储位置 | 例子 |
|---|---|---|
| 索引（轻量、可被 jq 查询）| `state.json.slices[sid].evidence` | commit SHA 列表、测试命令字符串、reviewer verdict（approved/issues_found）、报告文件路径 |
| 长文本内容（人读、可附图） | `evidence/<sid>/*.md` | subagent 完整报告、reviewer 列出的所有 issue 详情、benchmark 输出全文 |

state.json 的 `evidence.implementer_report` 字段值应是 `"evidence/s01/implementer-report.md"`（路径），而非内容本身。这样 state.json 保持紧凑、可程序化处理；详细叙述放在独立文件，git diff 友好。

### 3.4 文件生命周期

| 文件 | 创建时机 | 修改主体 | archive 时 |
|---|---|---|---|
| `changes/<id>/brief.md` | `/clarify`（L0 可选，L1+ 强制） | `/clarify`、`/escape` 触发的局部回填 | 归档 |
| `changes/<id>/spec.md` | `/spec` | `/spec`、`/escape` | 合并到 `specs/<capability>/spec.md`，归档原文件 |
| `changes/<id>/slices.md` | `/slice` | `/slice`、`/implement`、`/escape` | 归档 |
| `changes/<id>/escapes.log` | 首次 `/escape` | append-only | 归档（永不删除，作审计证据）|
| `changes/<id>/state.json` | `/clarify` 或 `/spec` 首次落盘 | 所有命令各维护对应字段；append-only 的子节点不删 | 归档 |
| `changes/<id>/evidence/<sid>/*.md` | `/implement` 每切片 | subagent / reviewer 各自写一份 | 归档 |
| `changes/<id>/verify-report.{md,json}` | `/verify` | 仅 `/verify` 写入，每次覆盖 | 归档 |
| `CONTEXT.md` | 任意时刻（懒创建） | `/clarify`、`/spec` | 不归档（项目级常驻）|
| `specs/<capability>/spec.md` | `/verify` 首次为 capability 归档 | `/verify` | 持续演进 |

---

## 4. 与三个原工作流的对照

### 4.1 命令对照表

| 阶段 | SliceSpec | OpenSpec | matt-skills | superpowers |
|---|---|---|---|---|
| 需求澄清 | `/clarify` | `/opsx:explore` | `/grill-me` + `/grill-with-docs` | `brainstorming` |
| 定义契约 | `/spec` | `/opsx:propose` 或 `/opsx:new` + `/opsx:continue` | `/to-prd` | (内嵌 brainstorm 的 design doc) |
| 任务拆分 | `/slice` | `/opsx:continue` (tasks 阶段) | `/to-issues` | `writing-plans` |
| 实现执行 | `/implement` | `/opsx:apply` | `/tdd` | `subagent-driven-development` |
| 范围漂移 | `/escape` | 无（需手动改 proposal） | 无（需修改 issue） | 无 |
| 一致性验证 | `/verify` | `/opsx:verify` + `/opsx:sync` + `/opsx:archive` | (依靠 PR review) | `finishing-a-development-branch` |

### 4.2 优势吸收来源

| 优势 | 来源 | 在 SliceSpec 中的体现 |
|---|---|---|
| Delta spec 语法 (ADDED/MODIFIED/REMOVED) | OpenSpec | `/spec` 自动识别 brownfield，生成 delta 块 |
| Scenario as Given/When/Then 桥接 | OpenSpec + final-workflow.md | spec.md 强制格式，分配稳定 ID |
| Requirement 粒度 merge | OpenSpec | `/verify` 的 sync 步骤 |
| Tracer-bullet 垂直切片 | matt-skills | `/slice` 的核心拆分原则 |
| HITL/AFK 分类 | matt-skills | slices.md 必填字段，决定 `/implement` 调度模式 |
| CONTEXT.md 领域语言 | matt-skills | `/clarify` 与 `/spec` 软依赖；不强制 |
| Soft vs Hard 依赖 | matt-skills | 所有命令在缺失上游产物时有 fallback |
| Subagent per task | superpowers | `/implement` 的 AFK 模式 |
| 两阶段 review (spec → quality) | superpowers | `/implement` 内嵌默认行为 |
| Plan as memory (单次读取) | superpowers | implementer subagent 从主会话接收 slice 全文 |
| Anti-rationalization | superpowers + final-workflow.md | 内嵌于 `/implement` 与 `/verify` 的 prompt |
| Bite-sized 任务粒度 | superpowers | slice → TDD cycle (2-5 min/cycle) |
| `brief.md` 持久澄清结论 | integrated-sdd-tdd 替代方案 | `/clarify` 落 8 节模板，L1+ 强制 |
| `state.json` 机器状态文件 | integrated-sdd-tdd 替代方案 | §3.3 schema，承载并行 owner / evidence / escapes 索引 |
| `write_scope` + `do_not_touch` 静态隔离 | integrated-sdd-tdd 替代方案 | slices.md 必填字段；`/implement` controller 强制 diff 校验 |
| `accepted_with_risk` review 结论 | integrated-sdd-tdd 替代方案 | state.json `reviews[].warnings_accepted_with_risk`，仅 Warning 适用 |
| 变更类型路由表 | integrated-sdd-tdd 替代方案 + final-workflow.md 第 11 节 | §5 新增（取代旧 L0/L1/L2/L3 单一梯度） |
| 否定 CLI flag 风格 | integrated-sdd-tdd 替代方案 | 全文删除 `--strict`/`--bulk`/`--dry-run`/`--review minimal`，改场景化自然语言 |
| `evidence/<sid>/*.md` 目录命名（vs `slices/<sid>/notes.md`） | integrated-sdd-tdd v2 | 命名更语义化；同时明确 state.json 装索引、文件装内容的边界 |
| HITL→AFK 降级路径 | integrated-sdd-tdd v2 提出（未细化） | SliceSpec 补判定主体、降级条件、`originally_hitl` 字段 |
| L0 路径归档不污染主 spec | integrated-sdd-tdd v2 提出（未规则化） | SliceSpec 补"`-l0` 后缀 + 跳过 spec sync"规则 |
| Strict 模式严重度语义 | integrated-sdd-tdd v2 提"strict"措辞（未定义）| SliceSpec 补"Warning 阻塞、Suggestion 仍提示"对照表 |
| 并行判定 5 步算法 + 物理 worktree 隔离 | integrated-sdd-tdd v2 仍 hand-wave | SliceSpec 把判定主体、5 步短路检查、共享资源黑名单写死 |

### 4.3 主动取舍掉的特性

| 特性 | 来源 | 取舍理由 |
|---|---|---|
| 单独的 `proposal.md` 与 `design.md` | OpenSpec | 合并入 `spec.md` 减少文件数；design 内容作为 spec 末尾可选附录 |
| `propose / new / continue / ff` 四种生成节奏 | OpenSpec | 合并为单个 `/spec`，由 LLM 内部决定一次性 vs 分段生成 |
| 单独的 `/sync` 命令 | OpenSpec | 99% 场景与 archive 同时发生，归入 `/verify` |
| 独立的 `/diagnose` 命令 | matt-skills | 作为非核心 skill 保留，不进 6 命令集 |
| `improve-codebase-architecture` 命令 | matt-skills | 同上，非核心任务流 |
| `grill-me` 与 `grill-with-docs` 二选一 | matt-skills | 合并为 `/clarify`，CONTEXT.md 存在与否自动切换行为 |
| 独立的 `tdd` 命令 | matt-skills + superpowers | TDD 是 `/implement` 的强制纪律，不单独成命令防止被跳过 |
| `executing-plans`（inline 批处理）| superpowers | `/implement` 默认 subagent，HITL slice 自然回到主会话，无需独立 inline 模式 |
| `finishing-a-development-branch` 独立步骤 | superpowers | 合并入 `/verify`（archive + PR 创建） |
| `using-git-worktrees` 显式命令 | superpowers | `/implement` 并行模式自动创建 worktree，不暴露给用户 |
| 完整代码块嵌入 `tasks.md` | superpowers | subagent 自己写代码不必预先嵌入，slices.md 保持高层 |
| 独立的 `/review` 命令 | integrated-sdd-tdd 替代方案 | review 是 `/implement`（双 reviewer）与 `/verify`（V1-V8）的内嵌行为，不单独成命令；独立 `/review` 会与内嵌 review 职责重叠并使命令数膨胀到 7 个 |
| 8 状态的 change 状态机 | integrated-sdd-tdd 替代方案 | 5 状态足以覆盖调度（draft/specified/implementing/verifying/archived），`escaping` 是 slice 级子状态，`clarifying` 与 `draft` 工程上等价 |
| 4 状态的 escape 状态机 | integrated-sdd-tdd 替代方案 | 二元 open/closed 足够；spec/plan/test-strategy 修订都是 closed 前的动作，不构成独立状态 |

### 4.4 净增量（三个原工作流都没有）

| 新增 | 价值 |
|---|---|
| `/escape` + `escapes.log` | final-workflow.md 第 10 节专设机制，三个原工作流都无显式实现。是 SDD+TDD 长期可维护的关键。 |
| 治理阈值 (escape>3 警告 / >slices/2 阻塞) | 量化"何时退回 clarify" |
| Scenario ID 与测试引用强校验 | OpenSpec 鼓励但未强制；matt-skills 完全没有；superpowers 完全没有 |
| 测试策略字段进入 slices.md | final-workflow.md 第 7 节系统化测试分层 |
| Controller `git diff` 边界校验作为结构性失败 | 三个原工作流都用 reviewer 自然语言判断越界，但子 agent 越界是结构性 bug 不是 review issue |
| Scenario ID 三种测试引用形式约定 | OpenSpec 提了 ID 但未规范引用，matt-skills/superpowers 完全没有 |
| `/verify` CI 退出码契约 + 报告路径约定 | 三个原工作流均未规范化 CI 集成接口 |
| 变更类型路由表 | final-workflow.md 第 11 节有，但三个原工作流都没有落到命令选择层 |

---

## 5. 上手路径与变更类型路由

### 5.1 治理强度梯度（L0 → L3）

#### L0：零配置即可用（首次试点）

仅使用 `/clarify` + `/implement`。
- 无需 `CONTEXT.md`、`brief.md`、`specs/` 目录。
- `/clarify` 结论留在会话上下文，不落 brief.md（L0 例外）。
- `/implement` 在缺 slices.md 时自动生成单切片占位（write_scope 取项目根），跑通后由用户事后补 spec。
- 目标：让团队习惯 TDD + subagent 派发，而不是先学治理。

**L0 路径的归档规则**（避免污染主 spec 仓库）：
- L0 模式下的 change 可执行 `/verify` 的"PR 前自检"模式（V1-V4 跳过，只跑 V5/V6 测试套件 + V7 git diff 自检）。
- archive 时移入 `changes/archive/YYYY-MM-DD-<id>-l0/`（后缀 `-l0` 显式标识）。
- **绝不执行 spec sync**：L0 的 change 没有结构化 spec.md，不写入 `specs/<capability>/`。
- 若 L0 change 事后被认为应进入主 spec，用户须先补齐 brief/spec/slices，重新运行 `/verify` 完整模式后 sync。这是单向流程，避免遗留 spec 污染。

#### L1：引入契约（团队稳定后）

加入 `/spec` 与 `/slice`，**强制 brief.md 与 state.json**。
- 开始维护 `specs/` 主仓库。
- 关键 Scenario 引入稳定 ID，测试中引用。
- `/verify` "PR 前自检"模式用作合并前 sanity check（非阻塞）。

#### L2：完整工作流

启用 `/escape` 与"严格校验"模式。
- CI 集成 `/verify`：退出码 ≥ 1 阻塞合并。
- 监控 escape 频率与漂移事件（来自 state.json `escapes[]`）。
- 引入 contract test 与跨服务校验。

#### L3：高治理 / 合规场景

参考 final-workflow.md 第 13 节：
- Spec/Test 自动追溯图（基于 Scenario ID）
- LLM 辅助语义检查（`/verify` 增加 V8 增强模式）
- 副作用与外部调用审计

### 5.2 变更类型路由表

不同变更类型走不同子路径（继承自 final-workflow.md 第 11 节，吸收自 integrated-sdd-tdd 替代方案第 12 节）。命令缺省时默认跳过。

| 变更类型 | 推荐路径 | 关键说明 |
|---|---|---|
| 新功能 / 重要业务行为 | `clarify → spec → slice → implement → verify` | 完整流程 |
| 外部 API / 数据契约变化 | `clarify → spec → slice → implement → verify`，**强制 contract test** | spec.md 中必含 External Contracts 章节 |
| Bug 修复（改变外部行为）| `clarify(轻量) → spec(补缺失 Scenario) → slice → implement → verify` | spec 必补 |
| Bug 修复（仅符合现有 Spec）| `implement → verify` | 直接 TDD 修复，不需要 spec 变更 |
| 内部重构（不改外部行为）| `implement(仅 refactor) → verify` | 不需要 spec.md；slices.md 可省略 |
| 性能优化（有外部承诺）| `clarify → spec(写入承诺) → slice → implement → verify` | 包含 benchmark 作为 evidence |
| 性能优化（仅内部）| `implement → verify(benchmark)` | 无需 spec |
| P0 热修复 | `implement(直接修+测试) → verify(轻量) → 补 brief/spec` | 先止血，事后限期补齐文档；state.json 标 `post-hoc-spec-pending` |
| 原型 / POC / 一次性脚本 | `implement` 单步 | 不进 archive 流程，不进入 `specs/` 主仓库 |

**路由判定**：
- 由 `/clarify` 在首轮对话中识别变更类型，写入 brief.md 的 Decision Log。
- 后续命令读 brief.md 判断是否需要跳过自己（如 brief 标 `type: internal-refactor` 时，`/spec` 自动跳过）。
- 用户可在任意命令调用时显式覆盖（"这次直接 implement"）。

---

## 6. 实施工程

### 6.1 文件目录约定

```
<project-root>/
├── CONTEXT.md                 # 可选，领域语言
├── docs/adr/                  # 可选，决策记录（继承 matt-skills）
├── specs/                     # 主 spec 仓库（archive 后合并目的地）
│   └── <capability>/
│       └── spec.md
└── changes/                   # 进行中的 change
    ├── <change-id>/
    │   ├── brief.md           # L1+ 强制；L0 可选
    │   ├── spec.md            # /spec 产物
    │   ├── slices.md          # /slice 产物
    │   ├── escapes.log        # /escape append-only
    │   ├── state.json         # 机器状态（参见 §3.3）
    │   ├── verify-report.json # /verify 产物（机器可读）
    │   ├── verify-report.md   # /verify 产物（人读）
    │   └── evidence/          # 子 agent 报告与 reviewer 长文输出
    │       └── <slice-id>/
    │           ├── implementer-report.md
    │           ├── spec-review.md
    │           └── quality-review.md
    └── archive/
        └── YYYY-MM-DD-<change-id>/
            └── (整个 change 目录归档，包括 state.json 与 escapes.log)
```

**为什么 `specs/` 在项目根而不在 `flow/specs/`**：避免占用顶层命名空间。`specs/` 是项目主资产（与 `src/`、`tests/` 同级），`changes/` 是临时工作目录可加入 `.gitignore` 子模式或保留在主分支。把工作流命名为 `flow/` 反而把流程绑死在工具层。

### 6.2 实现技术栈（参考）

- **命令分发**：作为 Claude Code skill 实现（每个命令 = 一个 SKILL.md + 模板文件）。
- **状态查询**：纯文件系统扫描（继承 OpenSpec 的设计哲学）；不需要数据库或外部服务。
- **subagent 派发**：使用 Agent 工具 + worktree isolation；prompt 模板复用 superpowers 的 implementer/reviewer 蓝本。
- **验证**：纯 Markdown 解析 + 简单 regex（继承 OpenSpec 的 validator 思路）；语义检查可选使用 LLM。

### 6.3 与 Claude Code 集成点

| Claude Code 能力 | SliceSpec 用途 |
|---|---|
| Skill 系统 | 6 个命令即 6 个 skill |
| Agent 工具 | `/implement` 的 subagent 派发 |
| Worktree | `/implement` 的并行隔离 |
| TaskList | 可选地把 slices.md 同步到 task 系统供监控 |
| Bash | `git` 操作、测试运行、CI 集成 |

---

## 7. 风险与边界

### 7.1 适用场景（继承 final-workflow.md 第 16 节）

- 长期维护的产品/平台
- 多人协作、跨模块变更
- 业务规则复杂、错误成本高
- AI 协作密集，需约束生成范围

### 7.2 不适用 / 应降级

- 一次性脚本、demo、POC：仅用 `/implement` 的 L0 模式，跳过 `/spec` 与 `/slice`。
- 需求每天剧变的早期探索：先停在 `/clarify` 反复迭代，不进入 `/spec`。
- 团队没有维护 spec 的能力或意愿：明确不引入 SliceSpec，使用项目原有流程。

### 7.3 已知风险

| 风险 | 缓解 |
|---|---|
| `/implement` 并行模式下 worktree 冲突 | `/slice` 阶段强制声明文件域（spec.md 的 capability 边界）；冲突时降级为串行 |
| subagent 调用成本（implementer + 2 reviewers per slice）| 提供 `--review minimal` 关闭 quality reviewer；用 Haiku 跑 mechanical implementer |
| Scenario ID 命名漂移 | `/verify` 检查 ID 唯一性与稳定性（archive 后 ID 不可变）|
| `/escape` 滥用导致 spec 反复修订 | 治理阈值（>3 警告，>slices/2 阻塞）；escape.log 作为 retrospective 证据 |
| spec.md 与 CONTEXT.md 术语不一致 | `/clarify` 与 `/spec` 默认读 CONTEXT.md；冲突时在对话中显式提示用户裁决 |
| state.json 与 slices.md/escapes.log 漂移 | 所有命令在退出前调用同一 state-writer 工具（先写 markdown，再用 markdown 解析回填 state.json 关键字段），不让命令各自手写 JSON |
| Scenario ID 误复用已 archive 的 ID | `/verify` 在 sync 阶段把已归档 ID 列入 reserved 集合；`/spec` 生成 ID 时查询并禁止冲突 |
| 子 agent 绕过 write_scope 写入仓库根（如 `.gitignore`） | `do_not_touch` 默认列入项目根文件 + 共享目录；controller 在 dispatch 前把 do_not_touch 注入子 agent prompt，作为硬约束而非建议 |

### 7.4 退出信号（继承 final-workflow.md 第 16 节）

监控以下信号，出现时退回探索 / 需求澄清 / 纯 TDD：

- 单 change 中 `/escape` > 一半 slice
- `/spec` 改动频率长期 > 实现改动频率
- `/verify` 反复无法收敛
- 团队把 spec.md 当流程负担而非意图契约
- 大量测试与实现细节耦合，重构频繁误伤

---

## 8. 一句话总结

> **SliceSpec = OpenSpec 的契约模型 + matt-skills 的切片纪律 + superpowers 的并行实现 + final-workflow.md 的语义边界裁决。6 个命令、3 个核心文件、零强制配置，从 L0 试点到 L3 合规线性升级。**

```
契约：  spec.md   （什么不能漂移）
切片：  slices.md （怎么端到端拆）
实现：  TDD by subagent （怎么证明做对）
逃生：  escapes.log （怎么知道偏离了多远）
归档：  verify → sync → archive （怎么收敛）
```

Spec 说清意图，Scenario 桥接证据，Slice 承载拆分，TDD 驱动实现，Escape 处理越界，Verify 保证语义一致。这套流程成立的关键不是文档数量也不是测试数量，而是三件事：

- Spec 是否说清真实业务意图。
- Test 是否提供足够可信的可执行证据。
- Code 是否在可维护结构下实现了这些行为。
