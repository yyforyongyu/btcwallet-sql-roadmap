# Task 433 — enforce the wallet sync test boundary

Component: **Wallet sync-state publication and integration-test harness**.

Type: **Test**.

Dependencies: **None**.

## Rationale

Public API integration cases can hide an invalid synchronization contract by
polling wallet-owned reads after the Wallet has already reported itself synced.
A retry can eventually succeed even if `Info.Synced` became true before all
transaction, UTXO, balance, and tip state was committed.

The Wallet and harness need one synchronization boundary. Once it succeeds,
wallet-owned reads for the reported tip must support deterministic, one-shot
assertions. Polling remains valid only for external transitions such as backend
peer synchronization, mempool delivery, and wall-clock lease expiration.

## Details

- Define the maintained invariant: when `Info.Synced` is true and
  `Info.SyncedTo` identifies the backend's exact best hash and height, every
  externally visible wallet view for that tip is committed and mutually
  consistent.
- Publish the synced state only after transaction, UTXO, balance, and synced-tip
  updates commit. Historical or stale notifications cannot advertise an
  inconsistent tip; preserve explicit reorganization handling.
- Strengthen `AssertWalletSynced` to require the sync flag and exact backend
  best hash and height. Include all observed values in a timeout failure.
- Make existing assertions over already-synced wallet state one-shot. Task 415
  and later public API itests consume this boundary when they land; they are
  not prerequisites or part of this task.
- Retain waits only for named external events. Mempool delivery and lease
  expiration helpers must say which event they await and remain bounded.
- Prohibit raw generic polling in `itest` scenario files with an automated
  source guard in the ordinary test target. Reject `wait.NoError`,
  `require.Eventually`, and equivalent retry loops there; allowed waits live in
  purpose-named `bwtest` helpers.
- Document the rule beside `AssertWalletSynced` and in the guard failure. Do not
  add a generic `WaitForWalletCondition`, change public sync semantics beyond
  the invariant, or redesign the chain backend or reorganization protocol.

## Suggested fix

Order the production sync transition after all wallet-owned state commits and
make `AssertWalletSynced` the sole chain-state retry boundary by comparing
`Wallet.Info` with the miner's exact best block. Convert existing downstream
wallet polling to direct assertions, move legitimate external waits behind
event-specific helpers, and add a small AST-based or equivalent source guard.

## Verification

- Mine to real kvdb, SQLite, and PostgreSQL Wallets, wait only through
  `AssertWalletSynced`, and check `ListUnspent`, `GetUtxo`, balance,
  transaction, and confirmation state exactly once.
- Prove `Info.Synced` stays false until wallet state and the exact synced tip
  are committed.
- Cover stale historical notification and explicit reorganization paths.
- Run the narrow Wallet sync and existing synchronized-state itests on every
  supported chain backend and all three database backends, without skipping
  kvdb inside the shared contract.
- Run the no-raw-polling guard through the ordinary unit-test target and prove
  a forbidden scenario call produces actionable guidance.

## Definition of done

- `AssertWalletSynced` proves the Wallet is synchronized to the backend's exact
  best hash and height after all wallet-visible state commits.
- The synchronization boundary has the same public contract on kvdb, SQLite,
  and PostgreSQL.
- Wallet-state assertions after that boundary execute once.
- Only bounded, purpose-named waits remain for genuinely external events.
- The ordinary test target rejects new generic polling in `itest` scenarios.
- Reorganization behavior remains covered and no generic polling escape hatch
  or unrelated chain redesign is introduced.
