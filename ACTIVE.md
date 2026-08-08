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

Dependency-ready tasks owned by different developers may proceed in parallel.
Multiple assignments to one developer are future ownership, not permission to
claim those tasks simultaneously.

| Task | Description | Phase | Assignee | Status | PR |
| --- | --- | ---: | --- | --- | --- |
| [Task 332][t332] | Define watch-only policy | 1 | YY + Gustavo | In progress | [#1305][pr1305], [#1306][pr1306] |
| [Task 333][t333] | Add PostgreSQL test harness | 1 | YY | In progress | [#1303][pr1303] |
| [Task 334][t334] | Define shared-chain ownership | 1 | YY | Assigned | — |
| [Task 335][t335] | Validate legacy migration snapshot | 1 | — | Unassigned | — |
| [Task 339][t339] | Define transaction visibility | 1 | — | Unassigned | — |
| [Task 342][t342] | Bound recovery windows | 1 | — | Unassigned | — |
| [Task 350][t350] | Use native PostgreSQL driver | 1 | YY | Assigned | — |
| [Task 352][t352] | Rotate kvdb passphrases atomically | 1 | — | Unassigned | — |
| [Task 354][t354] | Define Vault secret ownership | 1 | — | Unassigned | — |
| [Task 355][t355] | Correct transaction size accounting | 1 | — | Unassigned | — |
| [Task 357][t357] | Validate transaction amounts | 1 | — | Unassigned | — |
| [Task 359][t359] | Revalidate database identity | 1 | — | Unassigned | — |
| [Task 360][t360] | Document kvdb adapter identity | 1 | — | Unassigned | — |
| [Task 368][t368] | Serialize Wallet lifecycle | 1 | — | Unassigned | — |
| [Task 369][t369] | Reject repeated unlocks | 1 | — | Unassigned | — |
| [Task 378][t378] | Translate account errors | 1 | — | Unassigned | — |
| [Task 379][t379] | Translate load errors | 1 | — | Unassigned | — |
| [Task 380][t380] | Translate address lookup errors | 1 | — | Unassigned | — |
| [Task 400][t400] | Define live-watch semantics | 1 | — | Unassigned | — |
| [Task 415][t415] | Test public UTXO contracts | 1 | Abdullahi | In progress | [#1307][pr1307] |
| [Task 416][t416] | Build SQLite benchmark fixtures | 1 | — | Unassigned | — |
| [Task 421][t421] | Reconcile passphrase commits | 1 | — | Unassigned | — |
| [Task 424][t424] | Define transaction integrity | 1 | — | Unassigned | — |
| [Task 425][t425] | Isolate transaction authoring | 1 | — | Unassigned | — |
| [Task 427][t427] | Test Controller credentials | 1 | Gustavo | Assigned | — |
| [Task 428][t428] | Test resync and rescan | 1 | Gustavo | Assigned | — |
| [Task 429][t429] | Test account mutations | 1 | Gustavo | Assigned | — |
| [Task 430][t430] | Test account queries | 1 | Gustavo | Assigned | — |
| [Task 431][t431] | Test address allocation | 1 | Gustavo | Assigned | — |
| [Task 432][t432] | Test address imports | 1 | Gustavo | Assigned | — |
| [Task 433][t433] | Enforce sync test boundary | 1 | — | Unassigned | — |
| [Task 434][t434] | Test transaction creation | 1 | Abdullahi | Assigned | — |
| [Task 435][t435] | Test transaction publication | 1 | Abdullahi | Assigned | — |
| [Task 436][t436] | Test transaction reads | 1 | Abdullahi | Assigned | — |
| [Task 437][t437] | Test transaction labels | 1 | Abdullahi | Assigned | — |
| [Task 331][t331] | Fence Manager lifecycle | 2 | YY | Assigned | — |
| [Task 340][t340] | Persist database identity | 2 | YY | Assigned | — |
| [Task 329][t329] | Commit SQL genesis atomically | 4 | YY | Assigned | — |

Set a claimed task to `In progress` before implementation begins and add its
PR when one is opened. Remove a completed task's row when its task file moves
to the archive. Keep all dependency-free tasks visible even when unassigned.

## Coordination needed

- Tasks 332 and 333 were already in progress under YY when the one-task rule
  was adopted. Task 332 is shared with Gustavo because [PR #1305][pr1305] is
  stacked on his [PR #1306][pr1306]. These are grandfathered claims; neither
  developer may start another task until their current claim clears.
- Merged btcwallet [PR #1300][pr1300] already covers Manager creation and
  loading plus baseline Controller lifecycle and `Info`. Tasks 427 and 428
  contain only the remaining Controller credential and resync/rescan coverage.
- Draft btcwallet [PR #1279][pr1279] is evidence for Task 354's design
  decision, not an accepted API or task dependency.
- btcwallet [PR #1307][pr1307] claims only test-only Task 415. Abdullahi's
  Tasks 434–437 are future assignments from [Issue #1250][issue1250], not
  parallel claims; he may start one after Task 415 completes.
- [Issue #1250][issue1250] records Gustavo's future Controller,
  AccountManager, and AddressManager itest ownership as Tasks 427–432. These
  are assignments, not parallel claims; Gustavo may claim one only after
  Task 332 and PR #1306 clear, then must add its branch and PR.
- Task 433 and Task 415 have no hard dependency on each other and may be
  implemented by different developers in parallel. Coordinate their shared
  integration-test harness edits before merge.

[pr1279]: https://github.com/btcsuite/btcwallet/pull/1279
[pr1300]: https://github.com/btcsuite/btcwallet/pull/1300
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
[t433]: tasks/433-wallet-sync-test-boundary.md
[t434]: tasks/434-transaction-creation-itests.md
[t435]: tasks/435-transaction-publication-itests.md
[t436]: tasks/436-transaction-read-itests.md
[t437]: tasks/437-transaction-label-itests.md
