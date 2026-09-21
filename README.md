# btcwallet SQL wallet roadmap

This private repository coordinates the remaining work on btcwallet's
maintained SQL-backed `wallet/` library.

## How to use this repository

For numbered work:

1. Read [AGENTS.md](AGENTS.md) for the workflow and task rules.
2. Read [QUEUE.md](QUEUE.md) for the delivery phases and open
   dependencies.
3. Check [ACTIVE.md](ACTIVE.md) for active tasks, claims, and conflicts.
4. Select one active task or a coherent dependency stack.
5. Claim it and any dependent tasks you plan to stack in
   [ACTIVE.md](ACTIVE.md) before implementation.
6. Implement one task per branch and PR in its owning repository, stacking
   dependent PRs when that advances the chain.
7. After merge, archive the task and refresh the state map directly on `main`.

Claiming a task does not claim its dependencies. A task may be claimed before
it becomes active, and its PR may be stacked on an open prerequisite. Hard
dependencies still control bottom-up merge order and completion.

Unplanned PRs need no retrospective task file. After a PR merges into
`btcsuite/btcwallet`'s `sql-wallet` branch, count it under QUEUE's `Done`
section as one ad hoc task unless it is already a numbered task's implementation
PR. See [ad hoc PR accounting](AGENTS.md#ad-hoc-pr-accounting) for deduplication
and counts. Ad hoc completion does not reduce the numbered backlog.

## Sources of truth

| Information | Source |
| --- | --- |
| Numbered-task type, scope, and complete dependency graph | Individual file under [tasks/](tasks/) |
| Ad hoc scope and completion evidence | Merged PR linked from [QUEUE.md](QUEUE.md#done) |
| Delivery phases, open dependencies, completed PRs, supersession, and deferral | [QUEUE.md](QUEUE.md) |
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
file. Each numbered task owns its type, scope, and dependency contract. QUEUE
groups the work into delivery phases, and `ACTIVE.md` separately owns live
execution state.

## Delivery milestones

Finish Phase 1's API/runtime production prerequisites and the pre-claim gap
check before starting the main lnd port PR. Phase 1 itests may proceed alongside
it; other later-phase work follows [QUEUE](QUEUE.md)'s current-focus rule.
Relevant fixes and passing behavioral evidence, not named test-task or
benchmark completion, gate enabling an affected path.
[Task 470](tasks/470-lnd-wallet-port.md) delivers the lnd port. A prior roadmap
update assigns remaining required gaps to existing or new tasks.

After Phase 1, Phase 2A's bugs/gaps and 2B's verification run in parallel.
Phase 3 migration and Phase 4 optional improvements may overlap them as
dependencies allow. Hardening starts only after both 2A and 2B
finish. Modern kvdb stays supported for the current and next release; Task
353's retirement is deferred. [QUEUE.md](QUEUE.md) owns placement and gates.

## Scope

The maintained public APIs under `wallet/` are the target. A Bug fix must name
the maintained public component, a reachable trigger, and the wrong public
result. Normal production corrections stay in `package wallet` and directly
owned `wallet/internal` implementations. Existing legacy and external types
may be consumed unchanged, but their packages do not become part of the task.

Do not expand an ordinary Wallet fix into `waddrmgr`, `wtxmgr`, `walletdb`, the
`wallet/txauthor`, `wallet/txrules`, or `wallet/txsizes` modules, CLI, RPC,
Loader, lnd, daemon production, or deprecated Wallet methods and tests. Tasks
357 and 401 are narrow historical exceptions, not precedent. If the required
behavior cannot stay Wallet-owned, revise the task or obtain explicit approval
before implementation. [Task 470](tasks/470-lnd-wallet-port.md) owns the
lnd port as an explicit exception.

An integration-test task owns the tested public contract. Test and CI files may
live elsewhere, but any production correction remains Wallet-owned. Fix bugs
revealed by required cases in the same task and implementation PR, keep the
regression coverage, and leave unrelated production changes out.
Production fixes include focused tests and update existing affected itests;
they do not wait for separate coverage tasks. Later tests exercise implemented
behavior. Reuse available fixtures without imposing test-file ordering.
Keep benchmark tasks measurement-only and record defects they expose as
separate implementation tasks.
Run shared public contracts on kvdb, SQLite, and PostgreSQL. If a backend
cannot represent a scenario, document that structural exclusion in its task.

## Creating or changing a numbered task

Start from [TASK_TEMPLATE.md](TASK_TEMPLATE.md), use the next number in the
[task catalog](tasks/README.md), and follow [AGENTS.md](AGENTS.md)'s signed
direct-to-`main` roadmap update workflow.

Earlier plans are preserved under [archive/](archive/) as evidence, not current
instruction.
