# 任务总览

<!--
这个文件只做一件事：让人和 agent 扫一眼就知道 task 级别的整体进度。
不要把 task 细节写在这里——细节在各自的 tasks/phase-N-xxx/task-XX-xxx.md 文件里，
这个文件只保留 checkbox + 一行摘要 + 链接。

这样设计是为了省 token：agent 不需要每次把所有 task 的细节都读进上下文，
只在真正要执行某个 task 时才打开那一个文件。phase 级别的进度看 03-phases.md，
不要在两个文件里重复维护同一份状态。

勾选规则：
- [ ] 未开始
- [~] 进行中（同一时刻理论上只有一个 task 是 [~]）
- [x] 已完成并通过自测
- [!] 阻塞，需要人工介入（在旁边简要写阻塞原因，详情见 phases/ 对应日志）
-->

## phase-1-<name>

- [ ] [task-01-<name>](./tasks/phase-1-<name>/task-01-<name>.md) — <一句话摘要>
- [ ] [task-02-<name>](./tasks/phase-1-<name>/task-02-<name>.md) — <一句话摘要>

## phase-2-<name>

- [ ] [task-01-<name>](./tasks/phase-2-<name>/task-01-<name>.md) — <一句话摘要>
