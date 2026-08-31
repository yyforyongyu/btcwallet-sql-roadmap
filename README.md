# btcwallet SQL wallet roadmap

This private repository coordinates the remaining work on btcwallet's
maintained SQL-backed `wallet/` library.

## How to use this repository

1. Read [AGENTS.md](AGENTS.md) for the workflow and task rules.
2. Read [QUEUE.md](QUEUE.md) for the five delivery phases and open
   dependencies.
3. Check [ACTIVE.md](ACTIVE.md) for active tasks, claims, and conflicts.
4. Select one dependency-complete active task.
5. Claim it and any dependent tasks you plan to stack in
   [ACTIVE.md](ACTIVE.md) before implementation.
6. Implement exactly one task per btcwallet branch and PR, stacking dependent
   PRs when that advances the chain.
7. After merge, archive the task and refresh the state map directly on `main`.

Claiming a task does not claim its dependencies. A task may be claimed before
it becomes active, and its PR may be stacked on an open prerequisite. Hard
dependencies still control bottom-up merge order and completion.

## Sources of truth

| Information | Source |
| --- | --- |
| Task type, scope, and complete dependency graph | Individual file under [tasks/](tasks/) |
| Delivery phases, open dependencies, completed PRs, and superseded replacements | [QUEUE.md](QUEUE.md) |
| Active tasks, claims, and in-progress PRs | [ACTIVE.md](ACTIVE.md) |
| Task catalog and next number | [tasks/README.md](tasks/README.md) |
| Task shape | [TASK_TEMPLATE.md](TASK_TEMPLATE.md) |
| Agent workflow | [AGENTS.md](AGENTS.md) |
| Historical work | [archive/](archive/) |
| Derived machine-readable graph | [DEPENDENCY_GRAPH.json](DEPENDENCY_GRAPH.json) |

`DEPENDENCY_GRAPH.json` is an inspection snapshot, not a source of truth. It
records its source commit and includes direct edges, transitive closures,
dependency components, and scheduling-impact counts.

Do not duplicate task types, dependencies, or the complete backlog in this
file. Each task owns its type, scope, and dependency contract, QUEUE groups the
work into delivery phases, and `ACTIVE.md` separately owns live execution
state.

## Scope

The maintained public APIs under `wallet/` are the target. Avoid voluntary
changes to RPC, deprecated wallet APIs, or unrelated daemon behavior.

An integration-test task owns the tested public contract. Fix bugs revealed by
its required cases in the same task and implementation PR, keep the regression
coverage, and leave unrelated production changes out.
Keep benchmark tasks measurement-only and record defects they expose as
separate implementation tasks.
Run shared public contracts on kvdb, SQLite, and PostgreSQL. If a backend
cannot represent a scenario, document that structural exclusion in its task.

## Creating or changing a task

Start from [TASK_TEMPLATE.md](TASK_TEMPLATE.md), use the next number in the
[task catalog](tasks/README.md), and follow [AGENTS.md](AGENTS.md)'s signed
direct-to-`main` roadmap update workflow.

Earlier plans are preserved under [archive/](archive/) as evidence, not current
instruction.
