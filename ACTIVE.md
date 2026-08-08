# Assignments and active work

This file is the live execution view. It includes every Phase 1 task from
[QUEUE.md](QUEUE.md) plus every later-phase task with an assignment or active
claim. QUEUE owns dependency phases; this table owns assignments, work status,
implementation PRs, and coordination.

An assignment may be recorded before dependencies finish. A task may move to
`In progress` only after every hard dependency is complete. Claiming a task
never claims its dependencies, and each developer may have only one task in
progress unless YY explicitly approves an exception.

`Complete` means the implementation PR is merged, or a required design or ADR
task is accepted and merged. Assigned, in-progress, approved, or passing work
is not complete.

| Task | Component | Phase | Assignee | Status | PR |
| --- | --- | ---: | --- | --- | --- |
| [Task 332][t332] | Manager, AccountManager, and AddressManager | 1 | YY | In progress | [#1305][pr1305] |
| [Task 333][t333] | Manager and integration-test harness | 1 | YY | In progress | [#1303][pr1303] |
| [Task 334][t334] | SQL wallet architecture documentation | 1 | YY | Assigned | — |
| [Task 335][t335] | Legacy migration source snapshot | 1 | — | Unassigned | — |
| [Task 339][t339] | TxReader | 1 | — | Unassigned | — |
| [Task 342][t342] | wallet configuration | 1 | — | Unassigned | — |
| [Task 350][t350] | PostgreSQL Store | 1 | YY | Assigned | — |
| [Task 352][t352] | LegacyWalletVault and waddrmgr passphrase rotation | 1 | — | Unassigned | — |
| [Task 354][t354] | ADR 0010 secret-use contract | 1 | — | Unassigned | — |
| [Task 355][t355] | TxCreator and the txsizes module | 1 | — | Unassigned | — |
| [Task 357][t357] | txauthor and TxCreator | 1 | — | Unassigned | — |
| [Task 359][t359] | Manager and SQL Store design | 1 | — | Unassigned | — |
| [Task 360][t360] | ADR 0013 and the modern kvdb adapter contract | 1 | — | Unassigned | — |
| [Task 368][t368] | Controller | 1 | — | Unassigned | — |
| [Task 369][t369] | Controller | 1 | — | Unassigned | — |
| [Task 378][t378] | AccountManager public error boundary | 1 | — | Unassigned | — |
| [Task 379][t379] | Manager Load public error boundary | 1 | — | Unassigned | — |
| [Task 380][t380] | AddressManager GetAddressInfo error boundary | 1 | — | Unassigned | — |
| [Task 400][t400] | ADR 0004 and ADR 0005 synchronization contract | 1 | — | Unassigned | — |
| [Task 415][t415] | UtxoManager integration tests | 1 | Abdullahi | In progress | [#1307][pr1307] |
| [Task 416][t416] | wallet benchmark infrastructure | 1 | — | Unassigned | — |
| [Task 421][t421] | WalletVault and SQL wallet-secret persistence | 1 | — | Unassigned | — |
| [Task 424][t424] | ADR 0006 transaction and UTXO integrity contract | 1 | — | Unassigned | — |
| [Task 425][t425] | Wallet transaction-authoring wrapper boundary | 1 | — | Unassigned | — |
| [Task 427][t427] | Controller lifecycle and credential integration tests | 1 | Gustavo | Assigned | — |
| [Task 428][t428] | Controller information and synchronization integration tests | 1 | Gustavo | Assigned | — |
| [Task 429][t429] | AccountManager mutation integration tests | 1 | Gustavo | Assigned | — |
| [Task 430][t430] | AccountManager query integration tests | 1 | Gustavo | Assigned | — |
| [Task 431][t431] | AddressManager allocation and lookup integration tests | 1 | Gustavo | Assigned | — |
| [Task 432][t432] | AddressManager import and script integration tests | 1 | Gustavo | Assigned | — |
| [Task 331][t331] | Manager | 2 | YY | Assigned | — |
| [Task 340][t340] | Manager and SQL Store | 2 | YY | Assigned | — |
| [Task 329][t329] | Manager SQL genesis and SQL Store | 4 | YY | Assigned | — |

Set a claimed task to `In progress` before implementation begins and add its
PR when one is opened. Remove a completed task's row when its task file moves
to the archive. Keep all dependency-free tasks visible even when unassigned.

## Coordination needed

- Tasks 332 and 333 were already in progress under YY when the one-task rule
  was adopted. Do not start another YY task until both grandfathered PRs
  merge; use one in-progress row thereafter.
- btcwallet [PR #1306][pr1306] overlaps Task 332 and has no separate roadmap task.
  Coordinate its test-only remainder after Tasks 332 and 333.
- Draft btcwallet [PR #1279][pr1279] is evidence for Task 354's design decision, not an
  accepted API or task dependency.
- Draft btcwallet [PR #1307][pr1307] claims only test-only Task 415. It does not claim
  production UTXO, lease, PSBT, or SQL-integrity tasks.
- [Issue #1250][issue1250] records Gustavo's future Controller,
  AccountManager, and AddressManager itest ownership as Tasks 427–432. These
  are assignments, not parallel claims; Gustavo may move only one row to `In
  progress` at a time and must add its branch and PR when claimed.

[pr1279]: https://github.com/btcsuite/btcwallet/pull/1279
[pr1303]: https://github.com/btcsuite/btcwallet/pull/1303
[pr1305]: https://github.com/btcsuite/btcwallet/pull/1305
[pr1306]: https://github.com/btcsuite/btcwallet/pull/1306
[pr1307]: https://github.com/btcsuite/btcwallet/pull/1307
[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250

[t329]: tasks/329-atomic-wallet-account-creation.md
[t331]: tasks/331-manager-wallet-lifecycle.md
[t332]: tasks/332-watch-only-script-policy.md
[t333]: tasks/333-postgres-manager-harness.md
[t334]: tasks/334-shared-chain-state.md
[t335]: tasks/335-kvdb-to-sql-migration.md
[t339]: tasks/339-transaction-visibility.md
[t340]: tasks/340-durable-database-identity.md
[t342]: tasks/342-recovery-window-validation.md
[t350]: tasks/350-native-pgx-postgres.md
[t352]: tasks/352-kvdb-combined-passphrase-rotation.md
[t354]: tasks/354-vault-owned-secret-resolution.md
[t355]: tasks/355-transaction-size-accounting.md
[t357]: tasks/357-transaction-amount-validation.md
[t359]: tasks/359-revalidate-database-identity-design.md
[t360]: tasks/360-document-kvdb-adapter-identity.md
[t368]: tasks/368-wallet-startup-shutdown.md
[t369]: tasks/369-repeated-wallet-unlock.md
[t378]: tasks/378-account-manager-error-boundary.md
[t379]: tasks/379-manager-load-error-boundary.md
[t380]: tasks/380-address-lookup-error-boundary.md
[t400]: tasks/400-live-watch-semantics-adr.md
[t415]: tasks/415-utxo-public-integration-tests.md
[t416]: tasks/416-sqlite-benchmark-fixtures.md
[t421]: tasks/421-sql-passphrase-commit-reconciliation.md
[t424]: tasks/424-transaction-utxo-integrity-adr.md
[t425]: tasks/425-separate-transaction-authoring-wrappers.md
[t427]: tasks/427-controller-lifecycle-credential-itests.md
[t428]: tasks/428-controller-sync-info-itests.md
[t429]: tasks/429-account-mutation-itests.md
[t430]: tasks/430-account-query-itests.md
[t431]: tasks/431-address-allocation-lookup-itests.md
[t432]: tasks/432-address-import-script-itests.md
