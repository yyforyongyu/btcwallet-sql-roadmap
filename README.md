# btcwallet SQL wallet roadmap

This private repository coordinates the remaining work on btcwallet's
maintained SQL-backed `wallet/` library.

## How to use this repository

1. Read [AGENTS.md](AGENTS.md) for the workflow and task rules.
2. Read [QUEUE.md](QUEUE.md) for the delivery phases and open
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
file. Each task owns its type, scope, and dependency contract, QUEUE groups the
work into delivery phases, and `ACTIVE.md` separately owns live execution
state.

## Delivery milestones

Finish Phase 1's public APIs, required runtime behavior, and basic itests before
starting the main lnd port PR. Phase 2's critical fixes, scenarios, and simple
benchmarks can then proceed alongside that port; relevant correctness evidence,
not benchmark completion, gates enabling an affected path.

Phase 3 converts kvdb wallets to SQL, Phase 4 handles remaining gaps and bugs,
and Phase 5 handles Signer/Vault hardening. Modern kvdb stays supported for the
current and next release. Task 353's retirement is deferred, not a conversion
requirement. [QUEUE.md](QUEUE.md) owns task placement and the precise gates.

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
before implementation.

An integration-test task owns the tested public contract. Test and CI files may
live elsewhere, but any production correction remains Wallet-owned. Fix bugs
revealed by required cases in the same task and implementation PR, keep the
regression coverage, and leave unrelated production changes out.
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
