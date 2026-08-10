# Task 428 — cover Controller resync and rescan contracts

Component: **Controller Resync and Rescan integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] assigns Gustavo the Controller integration-test lane.
Merged [PR #1300][pr1300] covers the baseline `Info` contract. `Resync` and
`Rescan` remain without real-wallet coverage for lifecycle gates, bounded
requests, completion, and durable results after reopen.

## Details

- Add public integration cases for `Resync` and `Rescan` against a real
  SQL-backed Wallet and deterministic regtest chain.
- Require SQLite and register one backend-neutral source for PostgreSQL after
  [Task 333][t333] enables that harness.
- Cover stopped and started calls, current and earlier valid boundaries,
  malformed or out-of-range requests, cancellation, and the public completion
  or progress contract actually exposed by each method.
- Mine deterministic history, wait with bounded deadlines, and compare public
  information and recovered wallet-visible state before and after a full
  reopen.
- Keep the PR test-only and avoid defining future recovery or live-watch
  semantics owned by Tasks 342, 363, and 397–402.
- Use `Info` only to observe synchronization preconditions and postconditions;
  do not duplicate its standalone coverage from PR #1300.
- Exclude lifecycle credentials, account/address APIs, RPC, direct Store reads,
  bitcoind parity scenarios, and production fixes.

## Suggested fix

Add one synchronization-focused Controller itest source that builds a short
deterministic chain, records public checkpoints, issues bounded resync/rescan
requests, and observes completion only through maintained public APIs.

## Verification

- Run the focused SQLite Controller synchronization case selection.
- After Task 333 merges, run the same cases on PostgreSQL without backend
  branches.
- Verify cancellation and failure paths terminate polling and release the
  Wallet, Manager, chain backend, miner, and SQL resources.
- Verify the PR is limited to integration-test and reusable harness files.

## Definition of done

- `Resync` and `Rescan` have representative public lifecycle, success,
  invalid-boundary, and cancellation coverage.
- Completed synchronization results remain observable after durable reopen.
- Assertions use a real SQLite Wallet and remain backend-neutral.
- No production behavior or future recovery policy is introduced.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[pr1300]: https://github.com/btcsuite/btcwallet/pull/1300
[t333]: 333-postgres-manager-harness.md
