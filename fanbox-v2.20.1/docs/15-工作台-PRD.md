# 15 · 案台（个人工作台）PRD

版本 v0.5 · 2026-11-15 · 状态：共创中（账本 v1 已冻结，待实测）
对外名称：案台（FanBox Workbench）；本文中的「工作台」指这个模块。
上游：[14-工作台-概念与用户](14-工作台-概念与用户.md)

本期形态决策：FanBox 内新增视图 / 项目内 Markdown / 一任务一终端 tab / 微信·飞书优先。

## 0. 本版相对上一版的关键修订

- **规模与范围**：日常 3-5 个活跃软件项目；新项目默认建账本，旧项目只导关键文档。
- **冻结账本 v1**：ID 作用域、attempt 目录、task.md 字段、产物与证据格式。
- **状态权威源**：`task.md` 是唯一事实源；在线由主进程、离线由 `wb` 原子写；agent 只能提议 `review`。
- **任务分两档**：`quick`（当天、单文件/小范围、不碰依赖/配置/迁移）与 `full`（跨天/跨会话/需审计），越界自动升级。
- **自动通过默认关**：按项目显式开启；非 git 项目永久禁用；验收命令在可见 tab 执行。
- **inbox 改成单条文件**：不再往一个 `inbox.md` 并发追加。
- **log.md 只追加**：编辑器只读；人的批注写 `notes.md`，避免自动保存覆盖。
- **优先级与阶段分开**：需求用 P0-P3，交付阶段用 S0-S4。
- **`wb` 双模式**：FanBox 在线走 IPC；离线走单进程原子写 + `.wb.lock`；approve 只认人（TTY 二次确认或手机一次性 nonce）。
- **模板冻结**：六类模板（项目/REQ/DEF/ADR/NOTE/ART）、状态枚举、task `links`、验收表。
- **attempt 生命周期**：只在打回/显式重开时 +1；暂停恢复与续跑不 +1。
- **S0 不含自动通过**：S0 只做手工验收与 evidence 骨架；auto pass 留到 S1。
- **quick 最小目录**：`task.md` + `attempts/01/log.md`，其余按需生成。
- **事件层是任务域投影**：复用现有 `termFacts`/hooks，不造第二套事实。
- **快捷键与平台**：避开现有占用；CLI 跨平台，Windows UI 跟随社区移植。

## 1. 定位与范围

**一句话**：为「一个人 + 多个 agent」的长期项目，提供一套本地优先的项目账本——用文档记录意图，用任务把意图交给 agent，用证据链验收，并把结论回写知识。

第一版范围：

- 单机、单人、本地。
- 日常规模：3-5 个活跃软件项目；工程上限按 20 个项目设计，留归档余量。
- 软件项目为主，账本格式对内容/设计项目同样可用。
- 接入 Claude Code / Codex；其他 agent 走同一份 brief。
- 手机端以微信为主，飞书只发不收。

## 2. 用户与场景

| 用户 | 核心动作 |
|---|---|
| 设计者（你） | 捕获、拆任务、下发、验收 |
| 回归者（一周后的你） | 10 分钟恢复上下文 |
| agent | 读 brief、执行、登记、提议完成 |
| 遥控者（手机端的你） | 看摘要、批准、回答 |

关键场景：

- S1 新决策 → 关联需求 → 拆任务 → 下发。
- S2 灵感 → 收件箱 → 归类 → 任务。
- S3 缺陷 → 任务 → 修复 → 验收 → 关闭。
- S4 离开一周 → 回归摘要 → 批准/继续。
- S5 长任务等待 → 手机回答 → 继续。
- S6 半年后查一个决策的原因。

## 3. 名词与数据模型

### 3.1 名词

| 名词 | 定义 |
|---|---|
| 项目 | 一个有独立目录和账本的工作单元 |
| 账本 | `.workbench/` 下的全部文档 |
| 知识 | 需求/缺陷/决策/创意/资料/产物，图上的节点 |
| 任务 | 一次下发给 agent 的契约，可含多次尝试 |
| 尝试 | 一次执行会话（attempt），对应一个终端 tab / session |
| 过程 | 事件流 + 时间线 + 关单纪要 |
| 验收 | 人对照验收标准确认完成的动作 |
| quick 任务 | 当天、单会话、单文件或小范围、不碰依赖/配置/迁移的小活 |
| full 任务 | 跨天、跨会话、需要证据与验收的任务 |

### 3.2 目录结构

```
<project>/
└── .workbench/
    ├── project.md                 # 项目卡
    ├── _ids.json                  # ID 高水位（进 git）
    ├── knowledge/
    │   ├── REQ-001-....md
    │   ├── DEF-001-....md
    │   ├── ADR-001-....md
    │   ├── NOTE-001-....md
    │   └── ART-001-....md         # 需要长期追溯的产物（可选）
    ├── proposals/                 # agent 的知识提议（进 git）
    │   └── NOTE-900-....md
    ├── tasks/
    │   └── T-001/
    │       ├── task.md            # 契约与当前状态（永久，进 git）
    │       ├── brief.md           # 给 agent 的信封（永久，进 git）
    │       ├── notes.md           # 人的批注（永久，进 git）
    │       ├── summary.md         # 关单纪要（永久，进 git）
    │       ├── attempts/
    │       │   ├── 01/            # 目录号 %02d，与 JSON 里的整数 1 对应
    │       │   │   ├── log.md         # 人读里程碑（永久，进 git）
    │       │   │   ├── events.jsonl   # 机器事件流（全量，不进 git；轮转后 events-YYYYMMDD.jsonl）
    │       │   │   ├── artifacts.md   # 本次尝试的产物索引
    │       │   │   └── evidence.json  # 验收证据（进 git）
    │       │   └── 02/ ...
    │       └── artifacts/         # 需要快照的产物（默认不进 git）
    └── index.json                 # 可删缓存
```

全局：

```
~/.fanbox/workbench/
├── inbox/
│   ├── NOTE-001-....md    # 一条灵感一个文件
│   └── _archive/          # 已归类的原文件
├── index.json             # 跨项目索引缓存
└── settings.json          # 工作台偏好
```

规则：

- `.workbench/.gitignore` 局部忽略 `index.json`、`tasks/*/attempts/*/events.jsonl`、`tasks/*/artifacts/`，不改用户仓库根 `.gitignore`。
- 除上述三项外，账本其余文件（含 `log.md`、`notes.md`、`summary.md`、`evidence.json`、知识文档、`_ids.json`、`proposals/`）都进 git。
- `_ids.json` 记录每种前缀的 `next`；ID 永不复用，删除/归档也不回退。
- `quick` 的最小目录只有 `task.md` + `attempts/01/log.md`；`events.jsonl`、`artifacts.md`、`evidence.json` 按需生成。
- `proposals/` 是 agent 唯一能写知识建议的地方；人确认后由 `wb` 落库到 `knowledge/`。提议里的 NOTE 从 900 起编号，避免与 knowledge/NOTE 冲突。
- 每一条 inbox 灵感就是一篇 `NOTE` 文档，命名空间 `inbox`；归到真实项目时分配新 ID，保留 `origin: inbox/NOTE-001`，原文件移入 `inbox/_archive/`。

### 3.3 文档模板与 frontmatter

类型与前缀：

| type | ID 前缀 | 说明 |
|---|---|---|
| project | 无（用 slug） | `project.md` 的 `id` 即项目 slug |
| requirement | REQ | |
| defect | DEF | |
| decision | ADR | |
| note | NOTE | 含 inbox 条目 |
| artifact | ART | 需要长期追溯的产物 |
| task | T | |

模板共六类：项目卡 + REQ/DEF/ADR/NOTE/ART；「创意」是 `type: note` + `kind: idea`，「资料」是 `type: note` + `kind: reference`。

ID 规则：

- `project_id` = 项目目录 slug，生成后不改；显示名可改。
- 项目内 ID 格式 `前缀-三位序号`，永不复用；`_ids.json` 记录高水位。
- 跨项目引用写 `project_id/ID`，例如 `fanbox/REQ-001`；本地引用裸写 `REQ-001`。
- 禁止「先写本地再迁移」；迁移必须显式改链接。

每类必填 `{id, type, title, project, status, created, updated}`；`links` 可选。状态枚举见 §3.4。

知识文档示例：

```yaml
---
id: ADR-003
type: decision
title: 导出长图走本地 canvas
project: fanbox
status: accepted
priority: P1
created: 2026-11-15
updated: 2026-11-16
tags: [export, canvas]
links:
  - rel: implements
    to: REQ-009
  - rel: supersedes
    to: ADR-002
---
```

任务额外字段：

```yaml
id: T-001
type: task
project: fanbox
title: 修复导出长图在 4K 屏上的空白条
status: ready
priority: P1
profile: full | quick
risk: low | normal
current_attempt: 1
attempt_count: 1
session_id: null
scope_allow: [src/export/**]
scope_deny: [package.json, package-lock.json, .github/**]
verify_timeout: 600
diff_base: null
approved_by: null
approved_at: null
revoked_at: null
revoked_reason: null
links:
  - rel: implements
    to: DEF-007
```

正文固定节：

- `task.md`：目标 / 验收 / 上下文 / 约束 / 当前进展 / 关联。
- `summary.md`：目标 / 做法 / 结果 / 坑 / 后续。
- `brief.md`：由 task.md + 关联知识生成，人可改。

验收写在正文，固定表格式；`kind=machine` 必须有命令，`kind=human` 用截图/描述：

```
## 验收
| ID | kind | 标准 | 命令 |
|---|---|---|---|
| A1 | machine | 导出测试通过 | `npm test -- export` |
| A2 | human | 4K 屏无空白条 | —（截图 artifacts/4k.png） |
```

`verify_timeout` 对整张表生效；`evidence.json` 按 ID 记录结果。

`artifacts.md` 固定五列：

```
| ID | path | kind | attempt | note |
|---|---|---|---|---|
| ART-001 | artifacts/4k.png | image | 01 | 4K 屏对比 |
```

ART ID 在 `artifacts.md` 中登记；只有需要长期讨论时才创建 `ART-001.md` 知识文档，二者共用同一 ID。`path` 相对任务目录；跨 attempt 引用写 `attempts/NN/...`。

解析规则：

- frontmatter 用受限 YAML 子集（标量、列表、一层映射、`links` 的 list<map>）；其他结构拒绝，解析失败在界面上标「需修复」，不静默忽略。
- 缺 ID、坏链接、非法状态由 `wb validate` 报错；`--fix` 只修可安全推导的项。

### 3.4 状态机

```
backlog（想做）
  → ready（该做：至少一句话目标 + 一句话完成标准）
  → running（agent 在执行）
      ⇄ waiting（等人决策/输入；reason=permission|input）
      ⇄ blocked（外部阻塞；只能人设）
      ⇄ paused（关 tab / 人暂停）
          → restoring（恢复中）→ running
  → review（agent 提议完成）
      → done（人验收通过）
      → done（auto：risk=low 且 §3.8 门槛全过，24h 可撤销）
      → running（review 中继续追问，同 attempt）
      → running（打回重做，新 attempt）
      ← review（24h 内撤销自动通过）
done → archived（归档）
未 done 的任务 → dropped
```

规则：

- `quick` 任务只要求一句话目标与一句话完成标准；`full` 任务要求完整验收。
- `quick` 只允许单文件或小范围、不碰依赖/配置/迁移；执行中越界，自动升级 `full` 并通知人。
- agent 只能通过 `wb task review` 提议完成，不能直接写 `status: done`。
- 人验收通过由 UI/手机一次原子写入 `{status, approved_by, approved_at}`。
- 自动通过只能发生在 `risk: low`、`profile: full` 且 §3.8 门槛全过；记录 `approved_by: auto` 与判定明细。
- 人可在 24 小时内撤销自动通过，`done(auto) → review`，证据保留，不自动回滚代码。
- `blocked` 没有 hooks 来源，只能人设；`waiting` 才有 permission/input 两个事件来源。
- `ended` 只在 running/restoring 时映射为 `paused`；review/waiting 不被进程退出覆盖。
- `done → archived` 是唯一终态归档；未 done 才能 `dropped`。
- 一个任务同一时间只允许一个活跃 attempt；重做开新 attempt。

attempt 生命周期：

| 事件 | attempt | session_id | diff_base | tab |
|---|---|---|---|---|
| 首次下发 | 1 | 新建 | 下发时快照 | 新 tab |
| 断线续跑 | 不变 | 保留 | 不变 | 原 tab |
| 关 tab / 人暂停 | 不变 | 保留 | 不变 | 旧 tab 可恢复 |
| 从暂停恢复 | 不变 | 保留 | 不变 | 优先原 tab，不行则新 tab 注入 session_id |
| 打回重做 | +1 | 清空 | 重置 | 新 tab，旧 tab 关闭只读 |
| 人显式重开（reset） | +1 | 清空 | 重置 | 新 tab，旧 tab 关闭只读 |
| 自动通过撤销 | 不变 | 保留 | 不变 | 无 |

每次 attempt +1 都必须新建 tab 并关闭旧 tab；续跑与普通恢复不 +1。

### 3.5 状态权威与写入矩阵

| 动作 | 发起者 | 落盘者 | 约束 |
|---|---|---|---|
| `status`（在线） | 主进程 / UI | 主进程 | agent 只能 `wb task review` 提议 |
| `status`（离线） | 人 / agent 的 start/pause/block/review | `wb` 原子写 + `.wb.lock` | approve 必须人 TTY；agent 不能 approve |
| `task.md` 目标/验收/scope | 人 | 主进程 / `wb` | 下发后 scope 只读 |
| `task.md` 当前进展 | 主进程 / 人 | 主进程 / `wb` | agent 不直接写 |
| `brief.md` | 主进程 | 主进程 | 刷新默认追加「变更」，`--force` 才重写 |
| `log.md` | agent / 人 | 主进程 / `wb task append` | 只追加 |
| `notes.md` | 人 | 编辑器 | agent 只读 |
| `artifacts.md` | agent | `wb` | 只追加，路径相对任务目录 |
| `evidence.json` | 主进程 / `wb task verify` | 主进程 / `wb` | agent 只读 |
| `summary.md` | 主进程生成骨架 | 人确认 / agent 起草 | done 时生成 |
| `project.md` | 人 | 编辑器 / `wb` | `auto_approve` 只能桌面端 UI 改 |
| inbox NOTE | 人 / agent | `wb` | agent 可捕获，不可改已有知识 |
| `proposals/` | agent | `wb propose` | 人确认后落库 |
| `events.jsonl` | hooks / 主进程 | 主进程 | 任务域投影，不写知识 |

- `wb` 双模式：FanBox 在线时优先走 IPC；离线时单进程原子写 + `.wb.lock`。两种模式互斥，不并发写。
- `wb task review` 只是提交提议，落盘者校验后写 `review`；agent 可发起 `start/pause/block/review`，不能 `done/approve/drop`。
- S0 的 `wb task close --approve` 只在没有 `FANBOX_TASK_ID` 且是交互式 TTY 时可用，并要求输入任务 ID 二次确认。
- 手机批准走服务端一次性 nonce + actor 记录；值守 agent 只能转达，不能批准。
- agent 写知识只能用 `wb propose` 写 `proposals/`；`wb new` 只允许人在 TTY 或 UI 调用。
- 任务下发后 `scope_allow/scope_deny` 只读；要改必须新 attempt 或退回 review，并重置 `diff_base`。

事件映射：

| hooks 状态 | 任务状态 | 条件 |
|---|---|---|
| working | running | 有活跃 attempt |
| needs_permission | waiting（reason=permission） | 当前为 running/restoring |
| needs_input | waiting（reason=input） | 当前为 running/restoring |
| done（Stop，且无待答问题） | review | 当前为 running |
| ended（进程退出/关 tab） | paused | 当前为 running/restoring |
| 无 hooks 的 agent | 手动模式 | 只记 log/events，不参与自动状态与 auto pass |

- 启动时全量校验一次；之后事件驱动；每 30 秒只做一致性抽查，不轮询改状态。
- 发现 `done` 但 `approved_by` 为空，回退 `review` 并记一条违规事件。
- 人只能做「覆盖标记」记录纠正原因，不直接改 hooks 原始事实。

### 3.6 关系词表

| rel | 含义 | 例子 |
|---|---|---|
| implements | 实现 | ADR → REQ |
| depends_on | 依赖 | TASK → ADR |
| supersedes | 推翻/取代 | ADR-003 → ADR-002 |
| produces | 产生 | TASK → ART |
| blocks | 阻塞 | DEF → TASK |
| relates_to | 相关 | NOTE → REQ |

限制 6 个；不够用先讨论，不加词。

- `supersedes` 不自动改旧文档状态，由 `wb validate` 提示「旧决策仍为 accepted，是否标 superseded」。
- 决策的 `implemented` 是派生属性：关联任务 done 时界面显示「已实现」，不写进状态词表。
- `origin` 是普通字段，不是第 7 个关系词；需要图谱边时用 `relates_to`。

### 3.7 项目卡

```yaml
---
id: fanbox
type: project
title: FanBox
status: active
auto_approve: false
diff_limit: 300
protected_paths: [package.json, package-lock.json, .github/**]
created: 2026-11-15
updated: 2026-11-16
---
```

正文固定五节：目标 / 范围（做什么、不做什么）/ 关键决策索引 / 当前重点 / 关联项目。

项目状态枚举：`active` / `paused` / `archived`。关键决策索引只列 ADR ID（置顶/收藏），不复制内容；其余关系由图派生。

`auto_approve` 默认 false，只能桌面端 UI 修改；`protected_paths` 在代码里的默认清单上追加，不能删默认项，`wb validate` 会硬校验。健康状态由界面派生显示，不回写 `project.md`。

### 3.8 验收与风险分级

风险只有两档：

- **low**：建任务时由人在桌面端显式勾选；agent 与手机都不能设置。
- **normal**：默认，必须人验收。

自动通过默认**全局关闭**，按项目设置 `auto_approve: true` 才开启。S0 不实现自动通过，只做手工验收与 evidence 骨架；以下是 S1 定义。

开启后必须同时满足九条：

1. `risk: low` 且 `profile: full`。
2. 项目是 git 仓库；非 git 项目永久禁用。
3. 验收表中所有 `machine` 项都有命令。
4. 验收表全部通过；命令在可见的「验收」tab 执行，stdout/退出码写入 `evidence.json`。
5. 没有触碰保护路径：依赖清单与锁文件、CI 配置、密钥与环境变量、数据库迁移、发布脚本、系统配置；项目可在 `project.md` 的 `protected_paths` 追加，不能删默认项。
6. 改动在 `scope_allow` 内，未触碰 `scope_deny`；越界即回人工。
7. 没有 waiting / blocked；没有未回答的问题。
8. 有可回滚快照：FanBox 本回合快照或 git commit；`diff_base` 用下发时快照，避免历史未提交改动混入。
9. diff 低于阈值：默认 300 行，按 `diff_base` 计算；排除 `dist/`、`build/`、`node_modules/`、`*.min.*`、二进制、`artifacts/`；项目可调。

任何一条不满足，回到人工 review。

建任务时 `scope_allow` 默认取关联知识所在目录；人只确认，不强制手填。门槛不过只标「需人工」，不额外发通知。

自动通过后：

- 记录 `approved_by: auto` 与九条判定明细，写入 `evidence.json` 的 `auto_pass` 字段。
- 手机收到「已自动通过，24 小时内可撤销」。
- 撤销后任务回到 review；`approved_by` 保留 auto，写 `revoked_at` 与 `revoked_reason`；summary 标「已撤销」，已实现派生回退；不删除证据，不自动回滚代码。
- `evidence.json` 写入前对 stdout 做密钥/敏感词过滤和截断；手机只传摘要。

`quick` 任务永远不参与自动通过。

### 3.9 过程三层

| 层 | 文件 | 内容 | 是否进 git | 默认展示 |
|---|---|---|---|---|
| 事件层 | `attempts/NN/events*.jsonl` | hooks 事件、改动文件、diff 统计、session id、时间戳 | 否 | 收在「原始事件」 |
| 时间线层 | `attempts/NN/log.md` | 里程碑、决策、卡点 | 是 | 是 |
| 关单层 | `summary.md` + `evidence.json` | 目标/做法/结果/坑/后续；验收证据 | 是 | 是 |

规则：

- 事件层是任务域投影：从现有 `termFacts`/hooks 抽取 task/attempt 子集，不造第二套事实；session/term id 引用现有记录。
- 事件层只记结构化事件，不存终端全文；用途是审计与问题定位，不承诺还原现场。
- `log.md` 只能通过 `wb task append` 追加；编辑器对账本下的 log 设为只读。
- 每条里程碑不超过 3 行：做了什么、证据路径、下一步或卡点；`wb validate` 对超长给 warning。
- 事件层按天或 20MB 轮转为 `events-YYYYMMDD.jsonl`；读取统一走 `wb events --tail`，跨轮转文件按时间合并。
- 关单 90 天后可清理事件层；`log.md`、`summary.md`、`evidence.json` 永久保留。
- 人的临时批注写 `notes.md`，不写 `log.md`。

## 4. 功能需求

### M1 项目账本与知识图谱

| 编号 | 需求 | 优先级 |
|---|---|---|
| M1.1 | 新项目默认建账本；旧项目手动加入并只导关键文档 | P0 |
| M1.2 | 知识读写：新建/编辑/归档，沿用现有 Markdown 编辑器 | P0 |
| M1.3 | 自动索引：扫描 frontmatter + 链接，增量更新 index.json | P0 |
| M1.4 | 项目总览：项目卡 + 当前重点 + 最近活动 + 健康提示 | P0 |
| M1.5 | 追溯视图：选中需求，列出 决策 → 任务 → 产物 链 | P1 |
| M1.6 | 局部图谱：以选中节点为中心显示 1-2 跳邻域，边带 rel | P1 |
| M1.7 | 全局地图：按类型/状态过滤，默认折叠孤立节点 | P2 |
| M1.8 | 健康检查：未实现决策、缺陷 open>14 天、running>2h 无事件、ready>7 天未动、孤儿文档、项目>30 天无活动 | P1 |
| M1.9 | 一致性修复：缺 ID / 坏链接 / 非法状态一键列出并修复 | P1 |
| M1.10 | 旧项目迁移：agent 扫描关键文档后出提案，逐条确认；不批量改原目录 | P1 |

每个视图只回答一个问题：

- 总览：这个项目是什么、现在重点是什么。
- 追溯：这个需求最后落到哪。
- 局部图谱：改这个决策会牵动什么。
- 健康：哪里断了。

健康检查结果只进回归摘要，每项目最多 1 条；不做常驻榜单。`quick` 任务不进健康检查。

### M2 待办与创意收件箱

| 编号 | 需求 | 优先级 |
|---|---|---|
| M2.1 | 快速捕获：快捷键/顶栏/终端/微信，3 秒生成一条 `inbox/NOTE-xxx.md` | P0 |
| M2.2 | 收件箱 triage：归类到项目，转 NOTE/REQ/DEF/TASK，保留 origin 链接 | P0 |
| M2.3 | 待办清单：按项目/状态/优先级过滤；默认「需要我 > 进行中 > 该做 > 想做」 | P0 |
| M2.4 | 任务板：待办 / 进行 / 待验收 三列，不做自定义列 | P1 |
| M2.5 | 停滞提醒：`ready` 超过 7 天未动，与 M1.8 同一阈值 | P1 |
| M2.6 | inbox 老化：7 天未 triage 进每周摘要；超过 20 条提示合并；支持批量归类 | P1 |

`inbox.md` 只作为索引视图，不作为写入目标。

### M3 任务下发与档案

| 编号 | 需求 | 优先级 |
|---|---|---|
| M3.1 | 新建任务：从需求/缺陷/创意/空白创建；full 必填目标、验收、scope；quick 只需两句 | P0 |
| M3.2 | 生成 brief：从 task.md + 关联知识自动生成，人可改 | P0 |
| M3.3 | 下发：项目目录新建终端 tab，注入 env 与 `wb` 绝对路径，agent 先读 brief | P0 |
| M3.4 | 过程登记：每 attempt 一份 events/log/evidence；quick 按需生成；关单一份 summary | P0 |
| M3.5 | 状态同步：按 §3.5 映射 running/waiting/review/paused，人可覆盖标记 | P0 |
| M3.6 | 等待与回答：指挥台加任务徽标；一句话直接送进对应 tab | P0 |
| M3.7 | 暂停与恢复：关 tab = paused；复用项目记忆的「续上」，brief 增量补差 | P1 |
| M3.8 | 验收：展示清单/产物/diff/证据；S0 手工验收，S1 起 low 按 §3.8 判定 | P0 |
| M3.9 | 关单纪要：生成 summary.md，结论回写知识 | P1 |
| M3.10 | 多尝试：按 §3.4 生命周期开 `attempts/%02d`，记录原因、结果、session_id | P1 |
| M3.11 | 并行任务：不同任务各自一个 tab；同一任务不并行 | P1 |
| M3.12 | quick 档：当天、单文件/小范围、不碰依赖/配置/迁移；只写 task.md + attempts/01/log.md，其余按需；越界自动升级 full | P1 |
| M3.13 | 验收命令：`wb task verify` 在可见终端执行验收表，写 evidence.json；S0 可用 | P0 |
| M3.14 | 知识提议：`wb propose` 写 proposals/，人确认后落库 | P0 |

brief.md 必须包含：

- 角色与任务 ID。
- 目标与非目标。
- 必读文档清单（相对路径）。
- 验收标准（可勾选，标注 machine/human）。
- 风险等级与自动通过条件（risk=low 时必填）。
- 过程要求：里程碑用 `wb task append`；产物登记 artifacts.md；卡住先问。
- 完成要求：用 `wb task review` 提议，不得自行 done；自动通过由 FanBox 按 §3.8 判定。

下发注入环境变量：

- `FANBOX_TASK_ID`
- `FANBOX_TASK_DIR`
- `FANBOX_TASK_ATTEMPT`
- `FANBOX_WB_ROOT`
- `FANBOX_WB_CLI`（`wb` 绝对路径，不依赖 PATH）

下发协议 v1：

1. 主进程在项目根新建 tab。
2. `wb task brief T-001` 生成/刷新 brief。
3. 按 agent 模板发送首条 prompt；失败则只开 tab 并提示人工粘贴。
4. agent 不支持交互式首条 prompt 时，提示用户把 brief 拖进终端。

### M4 手机遥控

| 编号 | 需求 | 优先级 |
|---|---|---|
| M4.1 | 微信为主入口，复用 ClawBot 链路；飞书只发摘要 | P1 |
| M4.2 | 今日摘要：需要我 / 运行中 / 待验收 | P1 |
| M4.3 | 批准与打回：`批准 T-001` / `打回 T-001：原因`，二次确认；自动通过通知可撤销 | P1 |
| M4.4 | 回答等待：文字回复，路由到对应 tab | P1 |
| M4.5 | 快速捕获：`记一条：...` 生成 inbox NOTE | P1 |
| M4.6 | 追问进度：`看 T-001`，值守 agent 从 log/evidence 生成 3 行摘要 | P1 |
| M4.7 | 下发：`把 NOTE-003 做成任务`，agent 起草 brief，人确认后下发 | P2 |
| M4.8 | 通知节制：复用在场判断、夜间静默、限流；新事件需白名单 | P1 |
| M4.9 | 值守触发：review、waiting 超 10 分钟、running 超 2 小时无事件、每日摘要 | P1 |
| M4.10 | 回归摘要：上次活跃之后的变化，按需或每日推送 | P2 |

首批交付：M4.3 批准/打回 + M4.4 回答等待；第二批：M4.6 追问进度 + M4.5 快速捕获。

前置：微信 `termControl` 与 `/api/agent` 的发送路径先合并，避免第三条「回复到 tab」通道。

### M5 全局总览

| 编号 | 需求 | 优先级 |
|---|---|---|
| M5.1 | 跨项目今日：需要我 / 运行中 / 待验收 / 收件箱 | P1 |
| M5.2 | 项目健康榜：可选，默认关；按「该处理」排序 | P2 |
| M5.3 | 回归摘要卡：离开超过 24 小时后首次打开显示 | P2 |
| M5.4 | 每周复盘：完成/打回/阻塞/新增知识，agent 起草，人校对 | P2 |

## 5. 关键流程

### 5.1 设计 → 任务 → 验收 → 回写

1. 写 ADR/REQ（可用模板；agent 可代笔，人确认）。
2. 从知识建任务，填目标、验收、关联；小活选 quick。
3. 生成 brief，人审。
4. 下发，开 tab。
5. agent 执行，hooks 记 events，里程碑用 `wb task append`。
6. 卡住 → waiting → 通知人。
7. agent `wb task review` 提议完成 → review。
8. 人看证据 → 通过或打回；S0 手工验收，S1 起 low 且门槛全过可自动通过。
9. 通过 → 主进程生成 summary 骨架 → agent 起草 / 人确认 → 回写知识（状态、链接、坑）。

### 5.2 手机批准

通知包含：任务 ID、标题、验收清单、证据摘要（1-2 张关键截图或 diff 片段）、产物路径；大文件不传。

截图必须由你点选，可裁剪/打码后再发送。

回复 `批准 T-001` → 二次确认 → done；回复 `打回 T-001：原因` → 新 attempt。

手机批准走一次性 nonce + actor 记录；值守 agent 只能转达，不能批准。

### 5.3 断线与恢复

- 断线 → 自动续跑（已有）；3 次失败 → waiting + 手机通知。
- FanBox 重启 → 从 task.md/events 恢复运行中列表；不自动重开 tab，标 `restoring`。
- 恢复统一走项目记忆的「续上」；任务只存 `session_id` 与 brief 增量，不新建恢复机制；恢复不 +1 attempt，打回才 +1。

### 5.4 灵感 → 归类

inbox 一条 → 选项目 → 转 NOTE；或直接转 REQ/TASK（需补目标与验收）。转项目时分配新 ID，保留 `origin: inbox/NOTE-xxx`，原文件移入 `inbox/_archive/`。

### 5.5 旧项目迁移

- 只导入关键文档：README、ADR、需求、缺陷、最近任务记录。
- agent 出提案，逐条确认；原文档不复制、不改名，账本只建链接与摘要。
- 先迁一个项目试点，再推广到其余 2-4 个。

## 6. 界面与交互

### 6.1 入口

侧栏新增一级视角「案台」，与文件浏览并列；切换不丢终端。

顶部：今日 / 收件箱 / 项目 / 运行中。

### 6.2 任务详情

中栏 tabs：任务 / 信封 / 时间线 / 产物 / 证据 / 知识。右栏终端保持可见；下发后自动定位到对应 tab。

「证据」tab 同时承担验收命令执行，保证可见、可中断。

### 6.3 知识文档

沿用现有 Markdown 编辑器；顶部显示类型、状态、关联 chips。

知识提案复用「AI 整理」的提案卡：逐条确认、可回滚。

### 6.4 图谱

局部图谱默认；节点按类型着色，边标 rel；点节点打开文档；支持状态过滤。

不引入重型图库，用 canvas/SVG 自绘，保持零依赖。

### 6.5 快捷键

| 操作 | macOS | Windows |
|---|---|---|
| 搜索 | `⌘K` | `Ctrl+K` |
| 打开案台 | `⌘⇧A` | `Ctrl+Shift+A` |
| 快速捕获 | `⌘⌥N` | `Ctrl+Alt+N` |
| 下发当前任务 | `⌘⌥D` | `Ctrl+Alt+D` |

避开已有占用：`⌘↵` 是「用编辑器打开」、`⌘⇧W` 是关窗、`Ctrl+Shift+I` 是 DevTools。快捷键可在设置里改；Windows 下 `wb` 提供 `.cmd` 包装。

### 6.6 空态

- 无账本：一键初始化 + 三行说明。
- 无任务：从收件箱或需求开始。
- 无图谱：先建两个文档并链接。
- 零文档新用户：不要求先读文档，首屏给「建项目 / 记一条 / 看示例」。

## 7. 非功能需求

- **本地**：仅回环，写接口带 token，不联网。
- **性能**：5k 文档冷启动索引 <3s；增量 <300ms；列表 <100ms。
- **规模目标**：日常 3-5 个活跃项目；工程上限按 20 项目 / 5k 文档 / 500 任务设计。
- **可靠**：原子写；ID 不复用；index.json 可删重建；坏 frontmatter 不拖垮全局。
- **可迁移**：纯文本 + 相对链接；换机器直接拷目录。
- **可读**：任何 agent 不装 FanBox 也能读 brief 和账本。
- **过程体积**：events 按天或 20MB 轮转；关单 90 天后可清理，log/summary/evidence 永久。
- **隐私**：账本与文件不出本机；手机通知只发文字摘要，截图需人点选、可裁剪/打码。README 的「数据不出本机」改成同口径。
- **平台前提**：CLI 与账本格式跨平台；Windows UI 需要可用的 Electron 桌面壳（社区移植或自行打包）。网页版只能读账本、看任务，不能下发终端任务。
- **跨平台**：快捷键给 Ctrl 映射；命令行统一 `/` 分隔、反斜杠归一；`.cmd` 包装。

## 8. 复用与改动点

| 现有能力 | 工作台用法 | 不重复造 |
|---|---|---|
| 项目记忆 / ▶续上 | 任务存 `session_id`，恢复走「续上」 | 不新建恢复机制 |
| 本回合 diff / 整回合回滚 | 验收证据与可回滚点 | 不新建 diff 视图 |
| 指挥台 / 离开时叫我 | 任务徽标与新事件类型 | 不新建等待列表 |
| always-on 值守 | 新增 review/waiting/停滞事件 | 不新建调度器 |
| 微信 termControl | 先与 `/api/agent` 合并，再上 M4 | 不写第三条发送路径 |
| AI 整理提案 UI | 知识提议 | 不新建提案系统 |
| 11 个 agent launcher | 下发模板按 agent 可配 | 不硬编码命令 |
| `termFacts` / hooks | events 任务域投影 | 不造第二套事实 |
| ⌘K 全局搜索 | 案台搜索路由到现有搜索 | 不建第二套索引 |
| 截图直通车 | artifacts.md 引用现有路径，不复制文件 | 不建第二份素材库 |
| Skills X-Ray | `workbench` skill 纳入健康检查 | 不另建 skill 管理器 |

代码改动点：

| 位置 | 改动 |
|---|---|
| server.js | 新增 `/api/wb/*`：scan/index/read/write/validate/append/verify/propose/events；复用现有 token 与路径校验 |
| electron/main.js | 下发时注入 env；绑定 pty id ↔ task/attempt，不信任客户端 env；hooks 事件带 task_id/attempt；监听 `.workbench` |
| electron/preload.js | 暴露 `fanboxWorkbench` |
| public/app.js | 案台视角、任务详情、图谱、捕获 |
| public/style.css | 案台布局与状态色 |
| skills/workbench/SKILL.md | agent 读写账本的唯一规范 |
| lib/always-on.js | 新事件类型与叫醒条件 |
| lib/notify.js | 任务事件文案与节流 |
| `wb` CLI | 单文件 Node 脚本（只用核心模块），随 FanBox 分发；Windows `.cmd`；必要时 `ELECTRON_RUN_AS_NODE` 兜底 |

- `.workbench/.gitignore` 局部忽略，不改用户根 `.gitignore`。
- 账本下 `log.md` 在编辑器里只读；`notes.md` 可编辑。
- 手机传图必须人点选并可裁剪/打码，同步更新 README 隐私表述。
- Windows UI 跟随社区移植；官方验收只覆盖 CLI 与账本格式。若当前 Windows 只能跑网页版，S1 的终端下发不可用。

## 9. 分期路线

### S0 · 规范先跑通（终端里就能用）

- 账本 v1 契约、六类模板、ID/关系词表、`_ids.json`。
- `wb` CLI：init / new / task / propose / brief / dispatch / start / append / verify / review / close / validate / events / index；支持在线 IPC 与离线文件模式。
- S0 不实现 auto pass；验收用 `wb task verify` 在可见终端执行，写 evidence 骨架。
- `workbench` skill：让 agent 会建、会读、会登记、会提议。
- 验收：不用 UI，也能从建账本 → 任务 → verify → close 跑完一个真实任务。

### S1 · UI 闭环

- 侧栏工作台、项目总览、待办列表、任务详情。
- 下发到终端 tab、hooks 登记、验收与 summary。
- 快速捕获 + inbox triage + quick 档。

### S2 · 图谱与健康

- 追溯视图、局部图谱、健康检查。
- 全局地图与过滤。

### S3 · 手机与值守

- 微信今日；首批批准/打回 + 回答等待，第二批追问 + 捕获。
- 值守触发与回归摘要。
- 飞书摘要。

### S4 · 跨项目

- 全局总览、项目健康榜、每周复盘。

## 10. 验收标准

工作台新增一套五角色；原 README 的五角色（重度 vibe coder / 审美设计师 / 零文档新用户 / 终端老兵 / 破坏性质量官）仍用于整体 UI 与体验验收。

| 角色 | 关键验收 |
|---|---|
| 设计者（你） | 建项目 30s；捕获 3s；下发 30s；brief 不用手写上下文 |
| 回归者（一周后的你） | 10 分钟内答出每个项目「做到哪/为什么/下一步/坑」 |
| agent（执行者） | 只读 brief 就能开工；知道何时 append、何时该问；不越权改 done |
| 审计者（质量官） | 每个 full done 任务都有目标、验收、产物、log、summary、evidence；无孤儿任务 |
| 遥控者（手机端的你） | 30 秒内看清一条待验收并批准/打回；无需打开电脑 |

红线：

- agent 自行把任务标记 done；自动通过只能由系统按 §3.8 门槛执行，并可在 24 小时内撤销。
- 账本必须依赖数据库或云端才能读。
- full 任务过程无任何证据，只剩一句「做完了」。
- 手机端要装新客户端才能用。
- quick 任务被强制走完整验收流程。

## 11. 不做什么

- 通用笔记 / 双链编辑器 / 白板。
- 图数据库 / 向量库 / 语义搜索基建。
- 甘特图、工时、资源、多人权限、评论。
- 云同步、账号、第三方凭证。
- agent 自动完成任务。
- 自定义字段 / 自定义工作流 / 插件系统。
- 移动端全功能客户端。
- 自动生成没人读的长纪要（短摘要优先）。

## 12. 已定与待验证

已定（2026-11-15）：

- 命名：案台（FanBox Workbench）。
- 形态：FanBox 内新增「案台」视角。
- 规模：日常 3-5 个活跃软件项目；工程上限 20 项目。
- 账本：项目内 `.workbench` Markdown，图谱从链接推导；新项目默认建，旧项目手动加入、只导关键文档。
- 执行：一任务一终端 tab。
- 手机：微信为主，飞书只发；首批批准/打回 + 回答等待，第二批追问 + 快速捕获。
- 图谱第一版：追溯列表 + 局部邻域。
- 知识写入：agent 提议制，人确认落库。
- 验收：默认人验收；低风险任务按 §3.8 门槛自动通过，24 小时可撤销；默认全局关，按项目开启。
- 过程：三层记录，events 不进 git，log/summary/evidence 永久。
- 任务分 quick/full 两档；quick 限单文件/小范围、不碰依赖/配置/迁移，越界自动升级 full。
- 入口：侧栏一级视角，顶栏只放入口按钮。
- 运行模式：`wb` 双模式（在线 IPC / 离线原子写 + `.wb.lock`）；approve 只认人（TTY 二次确认或手机 nonce）。
- 模板：六类模板、状态枚举、任务 `links`、验收表冻结。
- 阶段：S0 不实现 auto pass；auto pass 从 S1 起按 §3.8 定义。
- attempt：按 §3.4 生命周期；只有打回/显式重开 +1，续跑与恢复不 +1。

待验证：

1. 低风险门槛的具体项与默认阈值：先按 diff ≤300 行、verify_timeout 600 秒、保护路径清单实测，再按项目调。
2. Windows 桌面壳是否可用：决定 S1 的终端下发在 Windows 上能否验收。

## 附录 A · 账本 v1 契约

### A.1 task.md 必填字段

| 字段 | 必填 | 说明 |
|---|---|---|
| id | 是 | `T-001`，项目内唯一 |
| type | 是 | `task` |
| project | 是 | 项目 slug |
| title | 是 | 一行标题 |
| status | 是 | 3.4 状态机，含 `archived` |
| priority | 是 | `P0`/`P1`/`P2`/`P3` |
| profile | 是 | `quick` / `full` |
| risk | 是 | `low` / `normal` |
| current_attempt | 是 | 当前 attempt 序号（整数） |
| attempt_count | 是 | 累计 attempt |
| session_id | 否 | 用于「续上」 |
| scope_allow | full 必填 | glob 列表 |
| scope_deny | full 必填 | glob 列表 |
| verify_timeout | 否 | 秒，默认 600，对整张验收表生效 |
| diff_base | 下发时写 | 下发时快照 id |
| approved_by | done 时写 | `human` / `auto` |
| approved_at | done 时写 | ISO 时间 |
| revoked_at | 撤销时写 | ISO 时间 |
| revoked_reason | 撤销时写 | 文本 |
| links | 否 | 与知识文档同格式 |
| created / updated | 是 | ISO 日期 |

`quick` 的最小字段集：`id/type/project/title/status/priority/profile/risk/current_attempt/attempt_count/created/updated`；`scope`、验收表按需补。

### A.2 events.jsonl 每行

```json
{"ts":"2026-11-15T14:03:00+08:00","task":"T-001","attempt":1,"source":"hook","type":"agent.waiting","session":"...","term":"...","data":{"reason":"permission"}}
```

`source`: `hook` / `human` / `system`。

| type | data |
|---|---|
| `agent.working` | `{}` |
| `agent.waiting` | `{"reason":"permission" \| "input"}` |
| `agent.review` | `{"evidence":"attempts/01/evidence.json"}` |
| `file.changed` | `{"files":[...]}` |
| `diff.stat` | `{"base":"...","added":42,"removed":8,"files":3}` |
| `acceptance.result` | `{"id":"A1","kind":"machine","exit":0,"ok":true}` |
| `status.change` | `{"from":"running","to":"review","actor":"agent"}` |
| `proposal.created` | `{"path":"proposals/NOTE-900.md"}` |

轮转文件名：`events-YYYYMMDD.jsonl`；`wb events --tail` 按时间合并多个文件。

### A.3 evidence.json

```json
{
  "task": "T-001",
  "attempt": 1,
  "accepted": ["A1"],
  "commands": [
    {"id":"A1","kind":"machine","cmd":"npm test -- export","exit":0,"stdout_tail":"...","at":"..."}
  ],
  "diff": {"base":"snapshot-20261115-1403","files":3,"added":42,"removed":8},
  "snapshots": ["round-20261115-1403"],
  "auto_pass": {"eligible": false, "checks": {}, "revoked_at": null}
}
```

`stdout_tail` 写入前过滤密钥/敏感词并截断到 2KB；`auto_pass.checks` 保存九条判定结果。

### A.4 index.json

```json
{
  "version": 1,
  "project_id": "fanbox",
  "generated_at": "2026-11-15T14:10:00+08:00",
  "nodes": [{"id":"ADR-003","type":"decision","title":"...","status":"accepted","path":"...","updated":"..."}],
  "edges": [{"from":"ADR-003","rel":"implements","to":"REQ-009"}],
  "tasks": [{"id":"T-001","status":"running","profile":"full","risk":"normal","priority":"P1","updated":"...","current_attempt":1}]
}
```

全局 index 只做缓存，带 `version` 与 `generated_at`，可随时删除重建；它是各项目 index 的聚合，额外带 `projects[]`（project_id、path、updated）。

- `path` 相对项目根，统一 `/` 分隔；跨项目 edge 的 `from/to` 写 `project_id/ID`。
- `nodes` 只放知识，`tasks` 单独放；task 的 `links` 生成 edges，不重复进 `nodes`。

### A.5 wb CLI v1

```
wb init [--project <slug>]
wb new <type> --title <text> [--project <slug>]        # 人 TTY / UI
wb propose <type> --title <text> --note <file>          # agent 写 proposals/
wb list [--project <slug>] [--status <s>] [--type <t>]
wb task new --title <text> --goal <text> --done <text> [--profile quick|full] [--priority P0-P3]
wb task brief T-001 [--force]
wb task dispatch T-001 [--agent claude-code|codex]
wb task start|pause|block|rework T-001
wb task append T-001 --line "..."                       # attempt 取 current_attempt
wb task verify T-001
wb task review T-001 --note "..."
wb task close T-001 --approve|--rework --note "..."
wb task promote T-001
wb validate [--fix]
wb events T-001 --tail 50
wb index [--all]
```

- `--attempt` 默认不暴露，取 `current_attempt`；只有补录才允许显式指定，`validate` 报 warning。
- `wb task new` 不接受 `--risk`；`risk: low` 只能由桌面端建任务 UI 设置。
- `wb task review` 读取当前 attempt 的 `evidence.json`；`--note` 可选。
- `wb task close --approve` 只在没有 `FANBOX_TASK_ID` 且是交互式 TTY 时可用，并要求输入任务 ID 二次确认。
- `wb` 双模式：FanBox 在线走 IPC；离线时单进程原子写 + `.wb.lock`。
- `wb` 是单文件 Node 脚本，只用核心模块；Windows 用 `.cmd` 包装，必要时 `ELECTRON_RUN_AS_NODE` 兜底。

退出码：0 成功，1 校验失败，2 用法错误，3 权限/路径错误。

### A.6 类型与状态枚举

| type | 状态 |
|---|---|
| project | active / paused / archived |
| requirement | draft / active / done / dropped |
| defect | open / fixing / verifying / closed / wontfix |
| decision | proposed / accepted / superseded / rejected |
| note | inbox / filed / promoted / archived |
| artifact | draft / final / archived |
| task | backlog / ready / running / waiting / blocked / paused / restoring / review / done / archived / dropped |

项目/知识必填 `{id, type, title, project, status, created, updated}`；task 见 A.1；`links` 可选。

### A.7 _ids.json

```json
{"version":1,"project_id":"fanbox","next":{"REQ":10,"DEF":8,"ADR":5,"NOTE":20,"ART":6,"T":21}}
```

`wb validate` 检查 `next` 大于现存最大序号；只能增，不能减。
