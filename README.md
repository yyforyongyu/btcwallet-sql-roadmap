# btcwallet SQL wallet roadmap

This private repository coordinates the remaining work on btcwallet's
maintained SQL-backed `wallet/` library.

## How to use this repository

1. Read [AGENTS.md](AGENTS.md) for the workflow and task rules.
2. Read [QUEUE.md](QUEUE.md) for the complete dependency phase map.
3. Check [ACTIVE.md](ACTIVE.md) for assignments, active PRs, and conflicts.
4. Select one dependency-complete task from the [task catalog](tasks/).
5. Claim it in [ACTIVE.md](ACTIVE.md) before implementation.
6. Implement exactly one task in one btcwallet branch and PR.
7. After merge, archive the task and refresh the state map through a roadmap
   PR.

Claiming a task does not claim its dependencies. A task may be assigned in
advance, but it cannot become active until every hard dependency is complete.

## Sources of truth

| Information | Source |
| --- | --- |
| Task scope and dependencies | Individual file under [tasks/](tasks/) |
| Dependency phases | [QUEUE.md](QUEUE.md) |
| Ownership and active PRs | [ACTIVE.md](ACTIVE.md) |
| Task catalog and next number | [tasks/README.md](tasks/README.md) |
| Task shape | [TASK_TEMPLATE.md](TASK_TEMPLATE.md) |
| Agent workflow | [AGENTS.md](AGENTS.md) |
| Historical work | [archive/](archive/) |

Do not duplicate task dependencies or the complete backlog in this file. Each
task owns its dependency contract, the queue derives the phase map, and
`ACTIVE.md` separately owns live execution state.

## Scope

The maintained public APIs under `wallet/` are the target. Avoid voluntary
changes to RPC, deprecated wallet APIs, or unrelated daemon behavior.

Keep integration-test tasks test-only and benchmark tasks measurement-only.
When either exposes a wallet defect, record a separate implementation task.

## Creating or changing a task

Start from [TASK_TEMPLATE.md](TASK_TEMPLATE.md), use the next number in the
[task catalog](tasks/README.md), and open a roadmap PR. Never push task or queue
changes directly to `main`.

Earlier plans are preserved under [archive/](archive/) as evidence, not current
instruction.
