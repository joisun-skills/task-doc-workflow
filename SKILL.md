---
name: task-doc-workflow
description: Provides a standardized, resumable documentation structure (docs/job/JOB-SLUG/) for executing non-trivial multi-step tasks — refactors, feature builds, migrations, investigations, or any work that could span multiple conversations. This skill has a hard dependency on the obra/superpowers skill suite (brainstorming, writing-plans, subagent-driven-development or executing-plans, test-driven-development, systematic-debugging, finishing-a-development-branch); it does not operate without them. Use this whenever the user starts a task that will take multiple steps or sessions, explicitly asks to "track progress", "记录任务进度", "整理任务文档", "断点续传", "写测试计划/测试用例", or when a task is complex enough that losing context mid-way would be costly. Also use when the user asks to set up project documentation conventions. Do not use for simple one-shot requests answerable in a single turn.
---

# 任务文档工作流 (task-doc-workflow)

## 这个技能解决什么问题

AI agent 在长任务里没有"外部记忆"——一旦对话中断或跨会话，agent 只能靠人复述状态。
本技能提供一套**固定的文件结构和状态标记规则**，让任意一次新对话都能通过读一个文件
恢复上下文。本技能**只负责文档持久化**，不负责"怎么想清楚一件事、怎么写代码、怎么调试"——
那些工作全部委托给 superpowers。

参考了社区里几个成熟实践并做了取舍：

- **[obra/superpowers](https://github.com/obra/superpowers)**：checkbox 作为断点续传依据、
  plan 里的 Global Constraints。本技能修正了它一个已知问题——大型 plan/task 文件被反复
  读入上下文浪费 token，这里改为每个 task 独立成文件；commit 粒度也参照它对齐到最小可验证单元。
- **Cline Memory Bank**：文件编号 + 分层依赖、"里程碑更新"而非频繁小改。本技能用单一的
  `03-phases.md` 状态面板替代"每次全读多个文件"，进一步省 token。
- **GitHub spec-kit**：进入下一阶段前的一致性检查、完成后的遗漏项回扫。

## 硬性前提：本技能依赖 superpowers，不做可选降级

**本技能的定位是 superpowers 的文档持久化层，不是独立可用的替代品。** 没有 superpowers，
本技能不工作——不会用自己的逻辑假装等价地把流程走完。

### 初始化时的依赖检查（在任何文档产出之前执行，只做一次）

1. 检查你当前是否能调用到 superpowers 的技能（尤其是 `using-superpowers` 这个引导技能，
   它是 superpowers 生效的开关）。不要假设可用性字段的具体名字或格式在所有平台上都一样，
   只需确认：这次会话里，superpowers 的技能是否实际可调用。
2. **可用** → 正常进入下面的工作流程。
3. **不可用** → 停下来，不做任何文档或代码改动，向用户说明本技能依赖 superpowers，并询问
   是否现在安装。安装方式按当前所在的 agent 环境判断，不要不假思索套用某一个平台的命令：
   - 如果在 Claude Code 里：`/plugin install superpowers@claude-plugins-official`
   - 如果在 Codex 里：通过 Codex 的插件市场安装，或克隆到 `.codex/skills/` /
     `~/.codex/skills/`，或使用 `npx skills add obra/superpowers`
   - 不确定所在环境、或以上都不适用：给出仓库地址 `https://github.com/obra/superpowers`，
     请用户参考 README 里对应自己工具的安装方式
   - 用户同意由你安装 → 执行对应安装步骤，验证成功后再继续
   - 用户想自己装 → 停下等待，用户装完确认后再继续
   - **用户拒绝安装 → 直接终止执行**。如果已经创建了 `03-phases.md`，把 `status` 设为
     `blocked`，`blocked_reason` 写"用户拒绝安装 superpowers 依赖"，然后停止，不产出
     任何后续文档或代码改动。

这个检查只在初始化时做一次；一旦确认 superpowers 可用，后面各阶段的委托都是**无条件的**，
不需要每次都重新判断"要不要用"。

## 用户信息收集与确认（按 agent 平台）

Discussion 阶段需要逐项收集需求和决策，其他阶段遇到范围签字、Plan 确认、阶段切换、执行授权等
明确的人类确认点时，也使用当前 agent 平台提供的结构化提问工具。一次只问一个高影响问题，
提供互斥且有实际意义的选项；收到答案后立即写回对应任务文档，再继续下一项。

### Codex：使用 `request_user_input`

在 Codex 中使用 `request_user_input`，不要把多个待确认问题改成普通文本清单让用户手动回复。
初始化本工作流时先确认该工具在当前会话是否可调用，并检查 Default mode feature：

```bash
codex features list
```

- `default_mode_request_user_input` 状态为 `true`：正常使用 `request_user_input`。
- 状态为 `false`：优先直接执行
  `codex features enable default_mode_request_user_input`；这会更新用户的 Codex
  `config.toml`。启用后明确提醒用户**重启 Codex 或重新开启对话**，当前会话不会热加载该能力。
- 无法修改配置：把上面的启用命令交给用户执行，记录当前确认项和恢复位置，待用户重新开启对话后
  从 `03-phases.md` 续接。
- feature 已开启但工具仍未暴露或调用仍报 unavailable：不要退回普通文本提问；提醒用户重新开启
  对话，并保留当前阶段状态。

### Claude Code：使用 `AskUserQuestion`

在 Claude Code 中使用 `AskUserQuestion` 收集 Discussion 决策和其他阶段的执行确认，同样遵循
一次一个问题、优先可选项、回答后立即落盘的规则。不要把 Codex 的 feature flag 或工具名套用到
Claude Code。

## 阶段契约表

进入下一阶段前，检查"输出"是否齐全、"禁止输出"是否还不存在——这是比散文描述更容易机械核对的门槛。

| 阶段            | 委托给（superpowers）                                                                                     | 输入                        | 完成后必须存在                                                      | 此时还不应存在                          | 下一阶段                                |
| --------------- | --------------------------------------------------------------------------------------------------------- | --------------------------- | ------------------------------------------------------------------- | --------------------------------------- | --------------------------------------- |
| 0 初始化        | using-superpowers（依赖门禁）                                                                             | 无                          | `03-phases.md`、`00-discussion.md`                                  | `01-plan.md`、`02-tasks.md`             | Discussion                              |
| 1 Discussion    | brainstorming                                                                                             | 用户需求                    | `00-discussion.md`（含范围确认+排除范围，人类已签字）               | `01-plan.md`                            | Plan                                    |
| 2 Plan          | writing-plans                                                                                             | 已确认的`00-discussion.md`  | `01-plan.md`（Global Constraints + Phase 划分，人类已签字）         | `02-tasks.md`、`tasks/`                 | Tasks                                   |
| 3 Tasks         | writing-plans 的输出（拆成独立文件）                                                                      | 已确认的`01-plan.md`        | `02-tasks.md` + `tasks/` 下各 task 文件 + `03-phases.md` 的阶段列表 | `phases/` 有实质内容                    | Execute                                 |
| 4 Execute       | subagent-driven-development 或 executing-plans，配合 test-driven-development；卡住时 systematic-debugging | 当前 task 文件              | `phases/phase-N-log.md` 追加记录 + 对应 commit                      | —                                       | Execute（循环）或全部完成后进 Test Plan |
| 5 Test Plan     | 本技能原创，superpowers 无对应产出                                                                        | `02-tasks.md` 全部 `[x]`    | `04-test-plan.md`                                                   | `05-test-cases.md`、`06-test-report.md` | Test Cases                              |
| 6 Test Cases    | 本技能原创                                                                                                | `04-test-plan.md`           | `05-test-cases.md`                                                  | `06-test-report.md`                     | Test & Report                           |
| 7 Test & Report | finishing-a-development-branch（收尾）                                                                    | `05-test-cases.md` 执行结果 | `06-test-report.md`，`03-phases.md` 的 `status = done`              | —                                       | 完成，或遗漏项转回 Tasks                |

## 目录结构

一个任务一个独立工作空间，路径固定为：

```
docs/job/<job-slug>/
├── 00-discussion.md             # 讨论与分析
├── 01-plan.md                   # 计划（含全局约束）
├── 02-tasks.md                  # 任务总览（task 级轻量 checkbox 清单）
├── 03-phases.md                 # 唯一入口：状态面板 + 阶段总览索引
├── 04-test-plan.md              # 测试计划（回归点清单）
├── 05-test-cases.md             # 测试用例矩阵
├── 06-test-report.md            # 测试报告（含遗漏项闭环）
├── phases/
│   └── phase-N-log.md           # 追加式执行日志（含 commit hash）
└── tasks/
    └── phase-N-<name>/
        └── task-XX-<name>.md    # 单个任务（一旦执行不再改内容）
```

编号 00-06 反映的是工作流阶段顺序，不是文件夹嵌套关系——`phases/` 和 `tasks/` 是
两个独立的顶层目录，分别装"多个 phase 的日志"和"多个 task 的详情"，不挂在某个编号
文件底下，这样 00-06 每一个都稳定是单个文件，翻目录时不会有的是文件、有的是文件夹。
`<job-slug>` 用 kebab-case，例如 `auth-migration`。是否加日期前缀由用户决定，不强制。

## Invariants（不变量）

这些规则任何时刻都必须成立，不因阶段而异：

1. **`02-tasks.md` 的 checkbox 是任务状态的唯一真相来源**，不是聊天记录，不是记忆。
2. **`03-phases.md` 是断点续传的单一事实来源**。任何新会话开始处理这个任务前，先读它，
   不要从对话历史推断进度——对话历史可能被压缩或丢失，文件不会。
3. **`tasks/` 下的 task 文件一旦创建、进入执行阶段就不可变**。要做什么写在 task 里；
   实际做了什么、有什么偏差，写进 `phases/` 的追加式日志，两者不混。
4. **`phases/` 下的日志只追加，不改写、不删除历史记录**。
5. **幂等性**：本技能的任何一步被重复执行，都不应该产生重复的 task、重复的日志条目、
   重复创建已存在的文件，也不应该覆盖用户已经手动编辑过的内容。执行前先检查目标文件/
   条目是否已存在，已存在则跳过或增量更新，不要无脑重新生成。

## Commit 规范

**Task 级别（多数 commit）**：

```
<job-slug>/phase<N>/task<NN>: <一行祈使句摘要，建议≤50字符>
<body>
#job-<job-slug>
#task-<NN>-<slug>
```

例：`auth-migration/phase1/task03: migrate auth middleware to token store`

`<job-slug>` 必须与 `docs/job/<job-slug>/` 的目录名一致。把 job 写入 subject，可以避免不同
job 中相同的 phase/task 编号在 Git 历史里发生歧义；body 中的 `#job-*` 和 `#task-*` 仍然保留，
用于稳定检索和机器化追溯。

如果一个 task 因为 TDD 红绿灯等原因拆成多个小 commit，**每个都带相同的前缀和 task 引用**，
这样 `git log --grep="#task-03-migrate-auth"` 能一次性拉出这个 task 的全部改动。

**里程碑级别（不绑定具体 task 的 commit，例如 plan 确认后的一次性提交）**：

```
job/<job-slug>: <里程碑描述>
```

例：`job/auth-migration: plan confirmed, entering task execution`

文档改动（`docs/job/` 下的文件）和代码改动**不做区分，混在同一批 commit 里**提交。

**追溯链路**：`phases/phase-N-log.md` 每条记录里写下对应的 commit hash（可以是多个），
形成 task 文件 ↔ 日志 ↔ commit ↔ 实际 diff 的完整链路，审查者不需要去猜"这个 task 改了什么"。

## 核心规则（按优先级分层）

**Critical（违反=流程失效，必须遵守）**

- 没有 superpowers 可用时不得继续工作（见"硬性前提"）。
- discussion → plan、plan → tasks 两个门槛必须有人类确认，不能由 agent 自行判断"应该没问题"就跳过。
- `tasks/` 下的 task 文件创建后不可变（见 Invariant 3）。

**Important（明显影响可追溯性/质量，默认应遵守，特殊情况可说明原因后偏离）**

- 每个 task 至少一次 commit，commit message 按“Commit 规范”执行，日志里记录 commit hash。
- 遇到非预期错误/库的怪异行为时，第一时间查官方文档/GitHub issue/社区讨论，不要第一反应就本地试错打补丁（见下方"外部资料优先原则"）。
- 测试报告发现的遗漏项转成新 task 追加进 `02-tasks.md` / `tasks/`，不当场顺手改代码了事。

**Preferred（值得做，但可以按情况灵活处理）**

- 变更记录只记里程碑，不记流水账。
- 小任务可以简化流程（见文末"何时可以简化"）。

## 失败回退规则

当前阶段掌握的信息不足以往下走时，**回退到上一阶段重新确认，而不是靠猜测继续**。
典型触发点：执行中发现需要一个架构层面的决策 → 回到 Plan 甚至 Discussion，而不是在
task 文件里硬编一个假设继续做。回退后，在 `phases/` 对应日志记录"为什么回退、回退到了哪一步"。

## 外部资料优先原则

遇到任何非预期的错误、库的怪异行为、框架报错——**第一次出现就先查官方文档、changelog、
GitHub issue/discussions，再动手改代码**，不是等本地试错几次都失败了才想起来查资料。
哪怕只花一次搜索的时间，也要在当次的 `phases/` 日志条目里留一句"查了什么、结论是什么"，
让这个判断本身可追溯。这一条和 `systematic-debugging` 的四阶段根因排查（偏内部代码层面的
证据收集）互补，不是重复——外部资料优先原则覆盖的是它没覆盖的"这是不是一个已知问题"。

## 工作流程（实现细节）

### 0. 初始化工作空间

依赖检查通过后：

```bash
mkdir -p docs/job/<job-slug>/phases docs/job/<job-slug>/tasks
cp assets/templates/03-phases.md docs/job/<job-slug>/03-phases.md
cp assets/templates/00-discussion.md docs/job/<job-slug>/00-discussion.md
```

先只创建 `03-phases.md` 和 `00-discussion.md`，其余文件在流程走到对应阶段时再从模板复制——
空文件没有信息量，只会造成"到底写了没有"的困惑（也呼应幂等性：不要一次性把所有阶段的
文件都创建出来，之后又要判断哪些是"真正开始了"哪些只是占位）。`03-phases.md` 的
"阶段列表"区块此时先留空，Plan 确认后再填。

若是代码类任务，委托 `superpowers:using-git-worktrees` 创建隔离的工作分支/worktree。

### 1. Discussion

委托 `superpowers:brainstorming` 做需求澄清和方案探讨，产出整理进 `00-discussion.md`
（模板：`assets/templates/00-discussion.md`），**额外补上**模板里的"范围确认"和
"明确排除的范围"两个区块——这是断点续传要用到、但 brainstorming 本身不产出的部分。
按上面的平台规则使用 `request_user_input` 或 `AskUserQuestion` 逐条澄清，一次问一个，
优先给选项而非开放式提问，每问完立刻记结论。
结束时必须有人类在"范围确认"区块签字，才能进入下一步。

### 2. Plan

委托 `superpowers:writing-plans` 生成实现方案（架构、技术栈选型、文件级拆分）。
它的 Global Constraints 概念直接复用；Phase/Task 拆分整理进 `01-plan.md`
（模板：`assets/templates/01-plan.md`）的 Phase 划分区块。同样需要人类确认才能往下走。
使用当前平台的结构化提问工具收集确认；确认后，把 Phase 列表同步填进 `03-phases.md`
的"阶段列表"（初始都是 `[ ]`）。

### 3. Tasks

`writing-plans` 已经把每个 task 的 Files/Interfaces 拆好了——这一步是把它输出里
"每个 task 一段"**物理拆分成独立文件**，不要沿用它"一个大 plan.md 装所有 task"的排布，
那正是本技能要修正的 token 浪费点。为每个 phase 建目录 `tasks/phase-N-<name>/`，
每个 task 用 `assets/templates/tasks/task-template.md` 建独立文件，同时维护
`02-tasks.md`（模板：`assets/templates/02-tasks.md`）作为轻量总览——只有 checkbox +
一句话摘要 + 链接，不放细节。

### 4. Action Steps Cycle & Record

实际的编码执行循环（含 TDD 红绿灯、两阶段 review）委托 `superpowers:subagent-driven-development`
或 `executing-plans`；卡住时委托 `systematic-debugging` 做根因分析。本技能只负责在每个
task 完成的前后做同步动作：

1. 读 `03-phases.md` 确认当前 phase/task。
2. 打开对应 task 文件（`tasks/phase-N-xxx/task-XX-xxx.md`），按验收标准执行
   （含委托技能跑的 TDD/review 循环）。
3. 完成后，在 `02-tasks.md` 把对应 checkbox 改成 `[x]`（或 `[!]` 并说明阻塞原因）。
4. 在 `phases/phase-N-log.md`（模板：`assets/templates/phases/phase-log-template.md`）
   追加一条记录：做了什么、commit hash、和计划是否有出入、验收结果、下一步。
5. 更新 `03-phases.md` 的 `current_task`、"现在在哪"、"下一步"；一个 phase 下全部
   task 完成后，把该 phase 的 checkbox 也改成 `[x]`。

### 5. Test Plan

全部 task 完成后，先核对 `02-tasks.md` 是否真的全部 `[x]`，执行中发现的偏差是否
都已同步回 `01-plan.md`。确认无误后写 `04-test-plan.md`（模板：`assets/templates/04-test-plan.md`）。
核心是系统性地想"这次改动可能破坏什么"，按影响面拆解：直接影响 / 间接影响（依赖方）/
边界情况 / 非功能性，每个回归点写清楚风险来源。

### 6. Test Cases

用 `05-test-cases.md`（模板：`assets/templates/05-test-cases.md`）。**每个回归点至少
对应一条用例**，且用例要能追溯回具体 task。用 P0/P1/P2 标优先级。

### 7. Test & Report Record

执行用例，结果记入 `06-test-report.md`（模板：`assets/templates/06-test-report.md`）。
测试中发现的、计划里没考虑到的问题（遗漏项），转成新 task 追加进 `02-tasks.md` /
`tasks/`，回到步骤 4 的循环，不当场顺手修。P0 用例全部通过才能视为完成；否则
`03-phases.md` 的 status 保持 `in_progress` 或标为 `blocked`。确认可交付后，委托
`superpowers:finishing-a-development-branch` 做测试验证、merge/PR 选择、worktree 清理，
再将 `03-phases.md` 的 status 更新为 `done`。

## 跨平台兼容性说明

本技能遵循 Agent Skills 开放标准的可移植核心（frontmatter + Markdown + `assets/` 子目录），
在 Claude Code、Codex 等支持该标准的平台上都能被加载和触发。但正文里两处内容做了
平台中立化处理，写新内容时延续这个原则：

- **判断"某个技能是否可用"时，不假设某个平台专属的字段名或检测方式**，只描述目标
  （"确认 superpowers 的技能是否实际可调用"），具体怎么检测交给当前所在的 agent 环境自己判断。
- **涉及安装命令、具体工具调用方式时按平台分支**，不要把某一个平台（如 Claude Code 的
  slash 命令）的写法当成唯一答案硬编码进指令里；不确定平台时，退回到给出通用信息
  （如仓库地址）而不是猜一个可能不适用的命令。
- **涉及用户信息收集和执行确认时也按平台分支**：Codex 使用 `request_user_input`
  （并确保 `default_mode_request_user_input` 已启用），Claude Code 使用
  `AskUserQuestion`。

## 何时可以简化

不是每个任务都需要走满 7 步。经验法则：

- 一次性的小改动、几分钟能说清楚的事，不要套这套流程，直接做（也不需要 superpowers 依赖检查）。
- 只有 1-2 个 phase、没有明显回归风险的任务，可以跳过 05/06/07（测试三件套），但
  `03-phases.md` + `00-discussion.md` + `01-plan.md` + `02-tasks.md` 建议保留。
- 是否使用本流程、用到哪一步，应该和用户确认，而不是默认套满全流程增加负担。
