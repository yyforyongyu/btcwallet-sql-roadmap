# Active work

This file separates planned ownership from work currently in progress. An
assignment does not authorize one developer to work on multiple tasks at once.

## Assigned tasks

- YY: [Task 329](tasks/329-atomic-wallet-account-creation.md),
  [Task 331](tasks/331-manager-wallet-lifecycle.md),
  [Task 332](tasks/332-watch-only-script-policy.md),
  [Task 333](tasks/333-postgres-manager-harness.md),
  [Task 334](tasks/334-shared-chain-state.md),
  [Task 340](tasks/340-durable-database-identity.md), and
  [Task 350](tasks/350-native-pgx-postgres.md).

## Active branches

Add a row before starting implementation and remove it after the PR merges.

- [Task 333](tasks/333-postgres-manager-harness.md): YY owns the PostgreSQL
  Manager and harness work on branch `task-postgres-harness-2` in [PR #1303].
- [Task 332](tasks/332-watch-only-script-policy.md): YY owns the watch-only
  and script policy work on branch `task-watch-policy` in [PR #1305].

## Coordination needed

- Gustavo's watch-only creation and itest work in [PR #1306] overlaps
  [Task 332](tasks/332-watch-only-script-policy.md). Coordinate before further
  wallet changes. It may become an itest-only follow-up after PRs #1303 and
  #1305.

[PR #1303]: https://github.com/btcsuite/btcwallet/pull/1303
[PR #1305]: https://github.com/btcsuite/btcwallet/pull/1305
[PR #1306]: https://github.com/btcsuite/btcwallet/pull/1306
