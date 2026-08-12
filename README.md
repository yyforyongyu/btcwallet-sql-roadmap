# btcwallet SQL wallet roadmap

This private repository coordinates the remaining work on btcwallet's
maintained SQL-backed `wallet/` library.

## How to use this repository

1. Read [AGENTS.md](AGENTS.md) for the workflow and task rules.
2. Read [QUEUE.md](QUEUE.md) for the five delivery phases and open
   dependencies.
3. Check [ACTIVE.md](ACTIVE.md) for active tasks, claims, and conflicts.
4. Select one dependency-complete active task.
5. Claim it in [ACTIVE.md](ACTIVE.md) before implementation.
6. Implement exactly one task in one btcwallet branch and PR.
7. After merge, archive the task and refresh the state map directly on `main`.

Claiming a task does not claim its dependencies. A task may be claimed before
it becomes active, but implementation cannot start until every dependency and
gate is complete.

## Sources of truth

| Information | Source |
| --- | --- |
| Task scope and complete dependency graph | Individual file under [tasks/](tasks/) |
| Delivery phases and open dependencies | [QUEUE.md](QUEUE.md) |
| Active tasks, claims, and in-progress PRs | [ACTIVE.md](ACTIVE.md) |
| Task catalog and next number | [tasks/README.md](tasks/README.md) |
| Task shape | [TASK_TEMPLATE.md](TASK_TEMPLATE.md) |
| Agent workflow | [AGENTS.md](AGENTS.md) |
| Historical work | [archive/](archive/) |

Do not duplicate task dependencies or the complete backlog in this file. Each
task owns its dependency contract, QUEUE groups the work into delivery phases,
and `ACTIVE.md` separately owns live execution state.

## Scope

The maintained public APIs under `wallet/` are the target. Avoid voluntary
changes to RPC, deprecated wallet APIs, or unrelated daemon behavior.

Keep integration-test tasks test-only and benchmark tasks measurement-only.
When either exposes a wallet defect, record a separate implementation task.

## Creating or changing a task

Start from [TASK_TEMPLATE.md](TASK_TEMPLATE.md), use the next number in the
[task catalog](tasks/README.md), and follow [AGENTS.md](AGENTS.md)'s signed
direct-to-`main` roadmap update workflow.

Earlier plans are preserved under [archive/](archive/) as evidence, not current
instruction.
