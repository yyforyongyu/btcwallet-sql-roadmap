# Task 455 — report exact wallet synchronization status

Component: **Wallet Controller synchronization information**.

Type: **Bug fix**.

Dependencies: **[Task 433][t433]**.

## Rationale

[Task 433][t433] makes the synchronized-state publication and itest boundary
atomic, but `Wallet.Info` still describes the Wallet as synced whenever the
Syncer is in its internal `synced` state. That state deliberately remains
unchanged while the Wallet catches up across a small chain-tip gap, so
`Info.Synced` can be true while `Info.SyncedTo` still identifies an earlier
block. A caller may reasonably interpret the boolean as proof that wallet data
reflects the current chain tip.

The same result exposes `IsRecoveryMode`, which collapses ordinary catch-up and
targeted rescanning into one boolean, and `RecoveryProgress`, which is always
zero. The public snapshot should report exact readiness and enough existing
state to explain why the Wallet is not ready without creating another source of
truth.

## Details

- Define `Info.Synced` as an exact public readiness result: the Syncer reports
  its existing synced state and the committed `SyncedTo` hash and height equal
  the chain backend's observed best hash and height.
- Include the observed chain tip and the Syncer's existing synchronization
  state in `Info`, using the current `backend-syncing`, `syncing`, `synced`, and
  `rescanning` vocabulary instead of adding overlapping booleans or another
  independently maintained state machine.
- Derive every field in one `Info` call from its owning source: the Store owns
  the committed wallet tip, the chain backend owns the observed chain tip, and
  the Syncer owns its activity state.
- Replace `IsRecoveryMode` and the placeholder `RecoveryProgress` directly
  when the final synchronization-state result makes them redundant. If genuine
  targeted-rescan progress is not available, omit it rather than manufacturing
  progress from the global wallet tip. Update maintained callers, mocks, and
  tests to the final shape without a compatibility adapter.
- Preserve `syncStateSwitchThreshold` and its operational behavior. A small gap
  may continue allowing ordinary wallet operations while `Info.Synced` reports
  false.
- [Task 428][t428] owns public integration assertions for the exported
  synchronization fields. This task does not add or revise integration-test
  source; [Task 433][t433]'s existing synchronization itest runs unchanged.
- Exclude RPC and deprecated-wallet changes, new polling helpers, recovery
  progress tracking, and any redesign of chain notification or rescan
  protocols.

## Suggested fix

Have `Wallet.Info` read the committed Store tip, the backend best block, and the
existing Syncer state, then derive the public fields from those values. Expose
the existing synchronization-state vocabulary through the result rather than
creating new state transitions or methods. Remove redundant recovery
projections instead of retaining parallel views.

## Verification

- Add focused `package wallet` Controller tests for exact match, height
  mismatch, same-height hash mismatch, backend syncing, ordinary syncing, and
  targeted rescanning.
- Prove a one-block gap below `syncStateSwitchThreshold` reports `Synced=false`
  while preserving the existing internal state and operation admission.
- Verify Store and chain-backend errors are returned with context and cannot
  produce a successful synchronization result.
- Run the focused Wallet package tests and Task 433's existing synchronization
  integration case unchanged on kvdb, SQLite, and PostgreSQL.

## Definition of done

- `Info.Synced` is true only for an exact committed wallet-tip and observed
  chain-tip match while the Syncer reports its synced state.
- Callers can inspect the wallet tip, chain tip, and synchronization activity
  without consulting internal state or inferring from recovery booleans.
- Redundant recovery projections are removed and no progress is invented.
- The threshold, notification, rescan, RPC, and deprecated-wallet behaviors are
  unchanged.
- The PR contains only this synchronization-information outcome.

[t428]: 428-controller-sync-info-itests.md
[t433]: ../archive/tasks/433-wallet-sync-test-boundary.md
