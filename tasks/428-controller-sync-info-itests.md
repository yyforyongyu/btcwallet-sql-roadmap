# Task 428 — cover Controller resync and rescan contracts

Component: **Controller Resync and Rescan integration tests**.

Type: **Test**.

Dependencies: **[Task 331][t331] and [Task 455][t455]**.

## Rationale

[Issue #1250][issue1250] assigns Gustavo the Controller integration-test lane.
Merged [PR #1300][pr1300] covers the baseline `Info` contract. `Resync` and
`Rescan` remain without real-wallet coverage for bounded requests, completion,
and durable results after reopen. At `sql-wallet` revision `48e7f4b`, the
[Controller][controller] acknowledges asynchronous scan transfer; method return
and live synchronization information do not prove historical completion.

## Details

- Add public integration cases for `Resync` and `Rescan` against real kvdb,
  SQLite, and PostgreSQL Wallets and a deterministic regtest chain.
- Register one shared public contract for all three database backends. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover calls through a Running Manager at current and earlier valid heights,
  invalid or out-of-range heights, missing rescan targets, and cancellation
  before admission. An accepted request finishes transfer independently of
  later caller cancellation; do not require that cancellation abort its scan.
- Observe recovered Wallet state with a bounded wait for the asynchronous scan.
  Method return or an already-true live `Info.Synced` is not scan completion;
  add no public progress API or removed recovery fields.
- Prove explicit historical rescan remains Wallet-scoped while live
  synchronization continues independently. Rescan activity cannot overwrite
  that Wallet's live readiness fields.
- [Task 472][t472] owns the SQL two-Wallet isolation assertion. This shared
  three-backend contract does not add multi-Wallet setup or duplicate that
  scenario.
- Mine deterministic history, wait with bounded deadlines, and compare public
  information and recovered wallet-visible state before and after constructing
  a fresh Manager and selecting the Wallet from its Start result.
- Do not redefine Task 342's caller-selected recovery window or future
  recovery/live-watch semantics owned by Tasks 397–402.
- Observe Task 455's exact live `Synced` and `SyncedTo` before, during, and
  after the scan through imported `wallet` APIs. Keep the public integration
  proof here without repeating PR #1300's standalone `Info` cases.
- Exclude lifecycle credentials, account/address APIs, RPC, direct Store reads,
  bitcoind parity scenarios, and unrelated production changes.

## Suggested fix

Extend the existing `itest/controller_test.go` with a short deterministic
chain, public checkpoints, and bounded resync/rescan requests. Observe
completion only through maintained public APIs; do not add a separate source
file or repeat the baseline Info cases.

## Verification

- Run the focused Controller synchronization cases on kvdb, SQLite, and
  PostgreSQL without backend branches.
- Verify exact live readiness remains independent of targeted historical
  work, and completed recovered state survives fresh Manager startup.
- Verify admission cancellation and failure paths terminate waits and release
  Wallet, Manager, chain backend, miner, and database resources.

## Definition of done

- `Resync` and `Rescan` have representative public success, invalid-boundary,
  missing-target, and admission-cancellation coverage.
- Completed synchronization results remain observable after durable reopen.
- Rescan state never overwrites the Wallet's live readiness.
- Assertions use real kvdb, SQLite, and PostgreSQL Wallets.
- No future recovery policy is introduced.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[pr1300]: https://github.com/btcsuite/btcwallet/pull/1300
[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t455]: ../archive/tasks/455-exact-wallet-sync-status.md
[t472]: 472-multi-wallet-rescan-isolation-scenario.md
[controller]: https://github.com/btcsuite/btcwallet/blob/48e7f4b7e0f3808b5022fd817d12c6a4361bb9c1/wallet/controller.go#L614
