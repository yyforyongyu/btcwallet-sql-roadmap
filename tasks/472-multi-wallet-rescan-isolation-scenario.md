# Task 472 — isolate historical rescans from live Wallets

Component: **multi-Wallet SQL rescan integration scenario**.

Type: **Test**.

Dependencies: **[Task 331][t331], [Task 415][t415], [Task 429][t429],
[Task 433][t433], [Task 436][t436], and [Task 455][t455]**.

## Rationale

A historical rescan in one Wallet must not disturb another Wallet's live
readiness or owned state. Per-Wallet rescan coverage cannot establish that
isolation. This outcome was split from [Task 449][t449] on 2026-09-19 so its
concurrent historical/live scenario can land independently of baseline
receive, spend, and reopen isolation.

## Details

- Create two Wallets with distinct accounts, addresses, and deterministic
  funding under one Manager and SQL database on the same network.
- Hold Wallet A's explicit historical rescan across a real live block while
  live synchronization continues. Prove Wallet B remains ready, receives live
  state in order, and exposes no Wallet A historical activity or owned state.
- Observe height and hash only through maintained public APIs. Shared chain
  facts may move for real live events, never merely because Wallet A scans
  historical data. Compare each Wallet's public synchronization information
  and expected owned transaction/UTXO sets at the relevant checkpoints.
- Reuse the existing Manager, miner, and exact synchronization fixtures. Do
  not add a scenario-specific coordinator, production fault seam, or direct
  Store inspection.
- Run the same scenario on SQLite and PostgreSQL. Do not register kvdb:
  its Manager supports only one Wallet per database and cannot represent this
  contract. Document that exclusion at registration, not a shared-case skip.
- [Task 428][t428] owns the shared one-Wallet rescan contract; Task 449 owns
  baseline cross-Wallet receipt, spend, and durable reopen. Neither test task
  is a prerequisite or a second coverage matrix here.
- Exclude separate Managers, cross-process fencing, network mismatch, stress,
  benchmarks, and changes to future recovery or chain-ownership policy.

## Suggested fix

Use disjoint public Wallet identities and existing deterministic chain controls
to keep the historical scan in flight across one live event. Assert the
observable isolation without depending on a private dispatcher design.

## Verification

- Run the focused scenario on SQLite and PostgreSQL, holding Wallet A's scan
  across a live block and checking Wallet B's readiness, ordering, and owned
  state without historical leakage.
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
