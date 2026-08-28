# Task 428 — cover Controller resync and rescan contracts

Component: **Controller Resync and Rescan integration tests**.

Type: **Test**.

Dependencies: **[Task 331][t331] and [Task 455][t455]**.

## Rationale

[Issue #1250][issue1250] assigns Gustavo the Controller integration-test lane.
Merged [PR #1300][pr1300] covers the baseline `Info` contract. `Resync` and
`Rescan` remain without real-wallet coverage for bounded requests, completion,
and durable results after reopen.

## Details

- Add public integration cases for `Resync` and `Rescan` against real kvdb,
  SQLite, and PostgreSQL Wallets and a deterministic regtest chain.
- Register one shared public contract for all three database backends. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover calls through a Running Manager, current and earlier valid boundaries,
  malformed or out-of-range requests, cancellation, and the public completion
  or progress contract actually exposed by each method.
- Mine deterministic history, wait with bounded deadlines, and compare public
  information and recovered wallet-visible state before and after a full
  reopen.
- Do not define future recovery or live-watch semantics owned by Tasks 342,
  363, and 397–402.
- Use `Info` to observe synchronization preconditions and postconditions.
  After [Task 455][t455], compare every exported synchronization field through
  the imported `wallet` package on all three backends. Task 455 owns the
  production result; this task owns its public integration assertions. Do not
  duplicate unrelated standalone `Info` cases from PR #1300.
- Exclude lifecycle credentials, account/address APIs, RPC, direct Store reads,
  bitcoind parity scenarios, and unrelated production changes.

## Suggested fix

Add one synchronization-focused Controller itest source that builds a short
deterministic chain, records public checkpoints, issues bounded resync/rescan
requests, and observes completion only through maintained public APIs.

## Verification

- Run the focused Controller synchronization cases on kvdb, SQLite, and
  PostgreSQL without backend branches.
- Compare every exported synchronization field through the imported `wallet`
  package before and after resync or rescan.
- Verify cancellation and failure paths terminate polling and release the
  Wallet, Manager, chain backend, miner, and database resources.

## Definition of done

- `Resync` and `Rescan` have representative public success, invalid-boundary,
  and cancellation coverage.
- Completed synchronization results remain observable after durable reopen.
- Assertions use real kvdb, SQLite, and PostgreSQL Wallets.
- No future recovery policy is introduced.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[pr1300]: https://github.com/btcsuite/btcwallet/pull/1300
[t331]: 331-manager-wallet-lifecycle.md
[t455]: 455-exact-wallet-sync-status.md
