# Assignments and active work

This file is the live execution view. It includes every Phase 1 task from
[QUEUE.md](QUEUE.md) plus any later-phase task that already has an assignee.
The queue owns dependency phases; this table owns assignments, work status,
and implementation PRs.

An assignment may be recorded before dependencies finish. A task may move to
`In progress` only after every hard dependency is complete. Claiming a task
never claims its dependencies, and each developer may have only one task in
progress unless YY explicitly approves an exception.

`Complete` means the implementation PR is merged, or a required design or ADR
task is accepted and merged. Assigned, in-progress, approved, or passing work
is not complete.

| Task | Component | Phase | Assignee | Status | PR |
| --- | --- | --- | --- | --- | --- |
| [Task 331][t331] | Manager and Controller | 1 | YY | Assigned | — |
| [Task 332][t332] | Manager, AccountManager, AddressManager | 1 | YY | In progress | [#1305][pr1305] |
| [Task 333][t333] | Manager and integration-test harness | 1 | YY | In progress | [#1303][pr1303] |
| [Task 339][t339] | TxReader | 1 | — | Unassigned | — |
| [Task 350][t350] | PostgreSQL Store | 1 | YY | Assigned | — |
| [Task 352][t352] | LegacyWalletVault, waddrmgr, kvdb AccountManager | 1 | — | Unassigned | — |
| [Task 355][t355] | TxCreator and txsizes | 1 | — | Unassigned | — |
| [Task 357][t357] | TxCreator, PsbtManager, txauthor | 1 | — | Unassigned | — |
| [Task 359][t359] | Manager and SQL Store design | 1 | — | Unassigned | — |
| [Task 360][t360] | ADR 0013 and modern kvdb adapter | 1 | — | Unassigned | — |
| [Task 340][t340] | Manager and SQL Store | 2 | YY | Assigned | — |
| [Task 334][t334] | Manager and Controller/Syncer | 3 | YY | Assigned | — |
| [Task 329][t329] | Manager SQL genesis and SQL Store | 4 | YY | Assigned | — |

Set a claimed task to `In progress` before implementation begins and add its
PR when one is opened. Remove a completed task's row when its task file moves
to the archive. Add a newly dependency-ready task when it moves into Phase 1.

## Coordination needed

- Tasks 332 and 333 were already in progress under YY when the one-task rule
  was adopted. Do not start another YY task until both grandfathered PRs merge;
  use one in-progress row thereafter.
- Gustavo's watch-only creation and itest work in [PR #1306][pr1306] overlaps
  [Task 332][t332]. Coordinate before further wallet changes. It may become an
  itest-only follow-up after PRs #1303 and #1305.

[pr1303]: https://github.com/btcsuite/btcwallet/pull/1303
[pr1305]: https://github.com/btcsuite/btcwallet/pull/1305
[pr1306]: https://github.com/btcsuite/btcwallet/pull/1306
[t329]: tasks/329-atomic-wallet-account-creation.md
[t331]: tasks/331-manager-wallet-lifecycle.md
[t332]: tasks/332-watch-only-script-policy.md
[t333]: tasks/333-postgres-manager-harness.md
[t334]: tasks/334-shared-chain-state.md
[t339]: tasks/339-transaction-visibility.md
[t340]: tasks/340-durable-database-identity.md
[t350]: tasks/350-native-pgx-postgres.md
[t352]: tasks/352-kvdb-combined-passphrase-rotation.md
[t355]: tasks/355-transaction-size-accounting.md
[t357]: tasks/357-transaction-amount-validation.md
[t359]: tasks/359-revalidate-database-identity-design.md
[t360]: tasks/360-document-kvdb-adapter-identity.md
