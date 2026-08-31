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

The public snapshot must report exact live readiness without exporting the
internal live-sync or historical-rescan state machine.

## Details

- Define `Info.Synced` from the committed Wallet tip, the observed shared or
  source tip, and the live delivery owner's state. It is true only when live
  processing is ready and the exact height and hash match.
- Keep only `Info.Synced bool` and truthful
  `Info.SyncedTo waddrmgr.BlockStamp` for this contract. Read the observed tip
  and raw source/delivery readiness from their owners in one consistent call,
  but keep those inputs private.
- Remove `IsRecoveryMode` and placeholder `RecoveryProgress` directly. Lnd
  already owns the configured recovery-window fact that selects startup
  recovery; ordinary catch-up and targeted rescans must not masquerade as that
  mode, and no progress field replaces the unsupported value.
- Starting, completing, or failing a targeted historical rescan cannot alter
  live readiness, another Wallet's readiness, or `SyncedTo`. Keep that state
  private unless a later concrete consumer proves a truthful public value is
  required.
- Add no observed-tip field, live-sync enum, historical-rescan field, polling
  helper, or compatibility view. Update maintained callers, mocks, and focused
  tests directly.
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

Have `Wallet.Info` compare one consistent committed Wallet tip with the private
observed ready tip and derive the existing public fields. Remove unsupported
recovery projections instead of exporting internal state or retaining
compatibility views.

## Verification

- Add focused table-driven `package wallet` tests that cross raw source and
  delivery readiness with exact live match, height mismatch, and same-height
  hash mismatch. Cover source error and independent targeted-rescan activity
  without asserting a public internal-state projection.
- Prove a one-block gap below `syncStateSwitchThreshold` reports
  `Synced=false` while preserving existing operation admission.
- Start and fail a targeted rescan while live tips remain matched and prove
  live readiness is unchanged. Repeat with a second Wallet and prove no rescan
  state or readiness leaks between them. Ordinary catch-up, configured startup
  recovery, and targeted rescans cannot produce a shared public recovery flag.
- Verify Store and source errors are returned with context and cannot produce a
  successful readiness result.
- Run focused Wallet package tests and Task 433's existing synchronization
  integration case unchanged on kvdb, SQLite, and PostgreSQL.

## Definition of done

- `Info.Synced` is true only for exact committed Wallet-tip and private
  observed-live-tip identity while live delivery is ready, and `SyncedTo`
  remains truthful.
- No observed-tip, live-sync, historical-rescan, recovery-mode, or recovery-
  progress field is exported by this task.
- A Wallet rescan cannot alter shared live-sync state, another Wallet, or the
  canonical frontier, and no progress is invented.
- Maintained callers use the final result without compatibility fields; RPC,
  notification, rescan protocol, and deprecated-wallet behavior are unchanged.
- The PR contains only this synchronization-information outcome.

[t428]: 428-controller-sync-info-itests.md
[t433]: ../archive/tasks/433-wallet-sync-test-boundary.md
[t334]: 334-shared-chain-state.md
