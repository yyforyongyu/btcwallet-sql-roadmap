# Active work

This file separates planned ownership from work currently in progress. An
assignment does not authorize one developer to work on multiple tasks at once.

An assignment may be recorded before dependencies finish. An active claim may
start only after every hard dependency is complete. Claiming a task never
claims its dependencies; other developers may complete them independently.

`Complete` means the implementation PR is merged, or a required design or ADR
task is accepted and merged. An assigned, active, approved, or passing
dependency is not yet complete.

## Assigned tasks

- YY: [Task 329](tasks/329-atomic-wallet-account-creation.md),
  [Task 331](tasks/331-manager-wallet-lifecycle.md),
  [Task 332](tasks/332-watch-only-script-policy.md),
  [Task 333](tasks/333-postgres-manager-harness.md),
  [Task 334](tasks/334-shared-chain-state.md),
  [Task 340](tasks/340-durable-database-identity.md), and
  [Task 350](tasks/350-native-pgx-postgres.md).

## Active branches

Add a row only after every hard dependency is complete and before starting
implementation. Remove it after the PR merges. Each developer may have only
one active row.

- [Task 333](tasks/333-postgres-manager-harness.md): YY owns the PostgreSQL
  Manager and harness work on branch `task-postgres-harness-2` in [PR #1303].
- [Task 332](tasks/332-watch-only-script-policy.md): YY owns the watch-only
  and script policy work on branch `task-watch-policy` in [PR #1305].

## Coordination needed

- Tasks 332 and 333 were already active under YY when the one-active-task rule
  was adopted. Do not add another YY claim; clear these grandfathered rows as
  their PRs merge and use one active row thereafter.
- Gustavo's watch-only creation and itest work in [PR #1306] overlaps
  [Task 332](tasks/332-watch-only-script-policy.md). Coordinate before further
  wallet changes. It may become an itest-only follow-up after PRs #1303 and
  #1305.

[PR #1303]: https://github.com/btcsuite/btcwallet/pull/1303
[PR #1305]: https://github.com/btcsuite/btcwallet/pull/1305
[PR #1306]: https://github.com/btcsuite/btcwallet/pull/1306
