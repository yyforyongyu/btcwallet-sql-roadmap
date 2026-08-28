# Task 455 — report exact wallet synchronization status

Component: **Wallet Controller synchronization information**.

Type: **Bug fix**.

Dependencies: **[Task 433][t433] and [Task 334][t334]**.

## Rationale

[Task 433][t433] makes synchronized-state publication and its integration-test
boundary atomic, but `Wallet.Info` can report synced while the committed Wallet
tip still trails the observed live chain. The same result collapses ordinary
live catch-up and targeted historical rescanning into one recovery projection
whose progress is not backed by real state.

The public snapshot must report exact live readiness and keep Wallet-scoped
historical rescan activity independent from Manager/shared live-sync state.

## Details

- Define `Info.Synced` from the committed Wallet tip, the observed shared or
  source tip, and the live delivery owner's state. It is true only when live
  processing is ready and the exact height and hash match.
- Keep `Info.Synced bool` and `Info.SyncedTo waddrmgr.BlockStamp`. Add
  `Info.ObservedTip waddrmgr.BlockStamp`, `Info.LiveSyncState LiveSyncState`,
  and `Info.HistoricalRescanActive bool`.
- Define exported `LiveSyncState uint8` with exactly
  `LiveSyncWaitingForSource`, `LiveSyncCatchingUp`, and `LiveSyncReady`.
  The current adapter or Manager coordinator exposes only raw source and
  delivery readiness. From one consistent snapshot, `Wallet.Info` derives the
  enum exactly once: waiting when the source is not ready; catching up when the
  source is ready but delivery is not ready or the tips differ; and ready only
  when both are ready and the tips match. `Synced` is true only in
  `LiveSyncReady` with equal `SyncedTo` and `ObservedTip` height and hash.
- Derive every field in one `Info` call from its owner. Store supplies
  `SyncedTo`, the shared source supplies `ObservedTip`, the current adapter or
  Manager coordinator supplies raw source and delivery readiness, and the
  Wallet supplies `HistoricalRescanActive`.
- Starting, completing, or failing a targeted historical rescan cannot change
  `LiveSyncState`, alter another Wallet's readiness, or overwrite `SyncedTo`.
  `HistoricalRescanActive` is true exactly from accepted targeted-rescan start
  through its completion or failure; it does not represent ordinary catch-up.
- Remove `IsRecoveryMode` and placeholder `RecoveryProgress` directly. No
  progress field replaces them because no authoritative progress source exists.
  Update maintained callers, mocks, and focused tests without compatibility
  fields or adapters.
- Preserve `syncStateSwitchThreshold` and its operational behavior. A small
  live gap may continue allowing ordinary Wallet operations while
  `Info.Synced` reports false.
- Keep the result owner-neutral across the current Syncer adapter and Task
  423's Manager coordinator so source cutover needs no second public shape.
- [Task 428][t428] owns public integration assertions for exported values.
  This task owns the production result and focused `package wallet` tests; it
  does not revise integration-test source.
- Exclude RPC and deprecated-wallet changes, new polling helpers, invented
  progress tracking, and notification or historical-rescan protocol redesign.

## Suggested fix

Have `Wallet.Info` read one consistent committed Wallet tip, observed live tip,
live delivery state, and independent Wallet rescan state, then derive the final
public result. Remove unsupported recovery projections instead of retaining
parallel or compatibility views.

## Verification

- Add focused table-driven `package wallet` tests that cross raw source and
  delivery readiness with exact live match, height mismatch, and same-height
  hash mismatch. Prove `Wallet.Info` alone derives every `LiveSyncState` value,
  and cover source error and independent targeted-rescan activity.
- Prove a one-block gap below `syncStateSwitchThreshold` reports
  `Synced=false` while preserving existing operation admission.
- Start and fail a targeted rescan while live tips remain matched and prove
  live readiness is unchanged. Repeat with a second Wallet and prove no rescan
  state or readiness leaks between them.
- Verify Store and source errors are returned with context and cannot produce a
  successful readiness result.
- Run focused Wallet package tests and Task 433's existing synchronization
  integration case unchanged on kvdb, SQLite, and PostgreSQL.

## Definition of done

- `Info.Synced` is true only for exact committed Wallet-tip and observed
  live-tip identity while live delivery is ready.
- `ObservedTip`, `LiveSyncState`, and `HistoricalRescanActive` have the exact
  exported types and ownership defined above, with no placeholder progress.
- A Wallet rescan cannot alter shared live-sync state, another Wallet, or the
  canonical frontier, and no progress is invented.
- Maintained callers use the final result without compatibility fields; RPC,
  notification, rescan protocol, and deprecated-wallet behavior are unchanged.
- The PR contains only this synchronization-information outcome.

[t428]: 428-controller-sync-info-itests.md
[t433]: ../archive/tasks/433-wallet-sync-test-boundary.md
[t334]: 334-shared-chain-state.md
