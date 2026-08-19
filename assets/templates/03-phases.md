---
job: <job-slug>
status: discussing          # discussing / planning / in_progress / testing / done / blocked
current_phase: null         # 例如 phase-1-xxx，未进入执行阶段前为 null
current_task: null          # 例如 task-03-xxx，未进入执行阶段前为 null
blocked_reason: null        # status=blocked 时必填，说明卡在哪、需要谁决策
created: <YYYY-MM-DD>
last_updated: <YYYY-MM-DD>
---

# <job-slug> 阶段总览 / 状态面板

> 这是本任务的**唯一入口**。文件编号是 03，但读取顺序永远排第一——
> 编号反映的是"工作流走到哪一步"，不是"该按什么顺序读"。
> 任何新会话开始时，**只需读这一个文件**（其余文档按需追溯），
> 就能知道：现在在哪个阶段/哪个 task、上一步做完了什么、下一步该干什么。

## 现在在哪

<!--
一句话说明当前处境，供人类和 agent 快速对齐。例如：
"计划已确认，正在执行 phase-1 的 task-03（迁移 auth 中间件），
上一个 task-02 已完成并通过自测，commit 见 phases/phase-1-log.md。"
-->

## 下一步

<!--
明确到"打开哪个文件、做什么"的颗粒度，不要写"继续开发"这种空话。例如：
"打开 tasks/phase-1-xxx/task-03-migrate-auth.md，按验收标准执行，
完成后在 phases/phase-1-log.md 追加一行记录（含 commit hash），
并把本文件 current_task 改成 task-04。"
-->

## 阶段列表

<!--
Plan 阶段确认后才填入，Discussion/Plan 阶段之前留空。
每个 phase 对应一份 phases/phase-N-log.md 执行日志；phase 的具体定义（目标/产出/依赖）
写在 01-plan.md 的 Phase 划分里，这里只做进度总览，不重复内容。

勾选规则：
- [ ] 未开始
- [~] 进行中（同一时刻理论上只有一个 phase 是 [~]）
- [x] 已完成（该 phase 下全部 task 都 [x]，且已过一致性检查）
- [!] 阻塞
-->

- [ ] [phase-1-<name>](./phases/phase-1-log.md) — <一句话摘要>
- [ ] [phase-2-<name>](./phases/phase-2-log.md) — <一句话摘要>

## 文档索引

| 文件 | 用途 | 状态 |
|---|---|---|
| [00-discussion.md](./00-discussion.md) | 讨论与范围确认 | |
| [01-plan.md](./01-plan.md) | 计划与全局约束 | |
| [02-tasks.md](./02-tasks.md) | 任务总览（task 级 checkbox 清单） | |
| [phases/](./phases/) | 各 phase 的执行日志 | |
| [tasks/](./tasks/) | 各 task 的详情文件 | |
| [04-test-plan.md](./04-test-plan.md) | 测试计划 | |
| [05-test-cases.md](./05-test-cases.md) | 测试用例矩阵 | |
| [06-test-report.md](./06-test-report.md) | 测试报告 | |

## 变更记录

<!--
只记"阶段性里程碑"，不记每次小改动。格式：日期 + 一行摘要。
例如：2026-07-09 计划确认，进入执行阶段
-->
- <YYYY-MM-DD> 任务创建
