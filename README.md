# Task Documentation Workflow

An Agent Skill that gives long-running engineering work a resumable,
Git-friendly documentation structure under `docs/job/<job-slug>/`. It keeps
scope, plans, task contracts, phase logs, test coverage, and verified lessons
connected without repeatedly loading one large planning document.

## Requirements

- An Agent Skills-compatible client
- The [`obra/superpowers`](https://github.com/obra/superpowers) skill suite

`obra/superpowers` is a hard dependency. This skill supplies the persistence
layer; it delegates brainstorming, planning, worktree setup, implementation,
debugging, TDD, review, and branch completion to the corresponding
`superpowers` skills.

## Install

Install `obra/superpowers` for your agent first. Then clone this repository into
an Agent Skills directory:

```bash
git clone https://github.com/joisun-skills/task-doc-workflow.git \
  ~/.agents/skills/task-doc-workflow
```

You can instead place it in a client-specific directory such as
`~/.claude/skills/task-doc-workflow` or
`~/.codex/skills/task-doc-workflow`.

To manage it inside another Git repository:

```bash
git submodule add \
  https://github.com/joisun-skills/task-doc-workflow.git \
  path/to/skills/task-doc-workflow
```

## What It Creates

The workflow creates documents only as their phases become active:

```text
docs/job/<job-slug>/
├── 00-discussion.md
├── 01-plan.md
├── 02-tasks.md
├── 03-phases.md
├── 04-test-plan.md
├── 05-test-cases.md
├── 06-test-report.md
├── lessons.md
├── phases/
└── tasks/
```

`03-phases.md` is the resumable status entry point, `02-tasks.md` is the task
state source of truth, task files are immutable contracts after execution
starts, and phase logs are append-only execution records.

See [SKILL.md](SKILL.md) for the complete workflow and
[assets/templates/](assets/templates/) for the document templates.

## Design Influences

The workflow builds on ideas from
[`obra/superpowers`](https://github.com/obra/superpowers), Cline Memory Bank,
and [GitHub spec-kit](https://github.com/github/spec-kit). Its task-per-file
layout and single status entry point are designed to reduce context reload cost
while preserving an auditable task-to-commit chain.

## License

Released under the [MIT License](LICENSE.txt).
