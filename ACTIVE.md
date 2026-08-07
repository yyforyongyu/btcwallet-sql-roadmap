# Active work

This is the shared claim ledger. Add a row before starting implementation, use
one row per task, and remove the row after the PR merges.

## Claimed tasks

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
