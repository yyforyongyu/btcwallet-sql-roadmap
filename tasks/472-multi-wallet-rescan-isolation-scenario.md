# Task 472 — isolate historical rescans from live Wallets

Component: **multi-Wallet SQL rescan integration scenario**.

Type: **Test**.

Dependencies: **[Task 331][t331], [Task 415][t415], [Task 423][t423],
[Task 429][t429], [Task 433][t433], [Task 436][t436], and [Task 455][t455]**.

## Rationale

A historical rescan in one Wallet must not disturb another Wallet's live
readiness or owned state. Per-Wallet rescan coverage cannot establish that
isolation. This outcome was split from [Task 449][t449] on 2026-09-19 so its
concurrent historical/live scenario can land independently of baseline
receive, spend, and reopen isolation. At `sql-wallet` revision `48e7f4b`,
[Rescan][source] only acknowledges asynchronous transfer and `Info` reports
live readiness; neither proves a historical scan is in flight. Task 423's
live fanout is required so Wallets do not compete for source notifications.

## Details

- Create two Wallets with distinct accounts, addresses, and deterministic
  funding under one Manager and SQL database on the same network.
- Prove Wallet A has entered historical chain work, hold it across a real
  live block, and check Wallet B's ready state and ordered live result before
  releasing A. B exposes no A historical activity or owned state. Then release
  A and verify its recovered public result; request return and live readiness
  alone are not evidence of scan entry or completion.
- Observe height and hash only through maintained public APIs. Shared chain
  facts may move for real live events, never merely because Wallet A scans
  historical data. Compare each Wallet's public synchronization information
  and expected owned transaction/UTXO sets at the relevant checkpoints.
- Reuse the real Manager, miner, and exact synchronization fixtures. A minimal
  test-only scan control may establish the overlap; it must leave B's live
  requests usable and release held work on failure so teardown can join. Add
  no production fault seam, progress API, coordinator, sleep-based overlap,
  changed chain contract, or direct Store inspection.
- Run the same scenario on SQLite and PostgreSQL. Do not register kvdb:
  its Manager supports only one Wallet per database and cannot represent this
  contract. Document that exclusion at registration, not a shared-case skip.
- [Task 428][t428] owns the shared one-Wallet rescan contract; Task 449 owns
  baseline cross-Wallet receipt, spend, and durable reopen. Neither test task
  is a prerequisite or a second coverage matrix here.
- Exclude separate Managers, cross-process fencing, network mismatch, stress,
  benchmarks, and changes to future recovery or chain-ownership policy.

## Suggested fix

Use disjoint Wallet identities and an existing chain control if available;
otherwise a minimal test-only entry/hold/release barrier around the real chain
fixture can establish the required interleaving through its existing interface.
Keep SQL/public assertions real and avoid prescribing a private helper or file.

## Verification

- On SQLite and PostgreSQL, prove historical entry precedes the live block,
  B processes it while A is held, and releasing A yields its recovered public
  result. Verify B's readiness, ordering, and owned state without leakage.
- Prove historical work alone cannot move the shared public chain facts.
- Verify the structural kvdb exclusion and complete joining of Wallet,
  Manager, miner, chain, and database resources on success and failure.

## Definition of done

Two SQL Wallets sharing one Manager preserve live readiness and owned-state
isolation while one performs a historical rescan. Evidence adds no baseline
spend/reopen matrix, cross-process claim, or new coordination architecture.

[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t415]: ../archive/tasks/415-utxo-public-integration-tests.md
[t429]: ../archive/tasks/429-account-mutation-itests.md
[t433]: ../archive/tasks/433-wallet-sync-test-boundary.md
[t436]: ../archive/tasks/436-transaction-read-itests.md
[t455]: ../archive/tasks/455-exact-wallet-sync-status.md
[t428]: 428-controller-sync-info-itests.md
[t449]: 449-multi-wallet-sql-isolation-scenario.md
[t423]: 423-manager-chain-source-cutover.md
[source]: https://github.com/btcsuite/btcwallet/blob/48e7f4b7e0f3808b5022fd817d12c6a4361bb9c1/wallet/controller.go#L643
