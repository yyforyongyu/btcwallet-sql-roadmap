# Task 455 — report exact wallet synchronization status

Component: **Wallet Controller synchronization information**.

Type: **Bug fix**.

Dependencies: **[Task 433][t433] and [Task 334][t334]**.

## Rationale

`Wallet.Info` can report synchronized while the committed Wallet tip trails
the observed live chain. It also exposes placeholder recovery fields that mix
ordinary catch-up with historical rescans without truthful backing state. Lnd
needs an exact readiness result, not a projection of private sync machinery.

## Details

- `Info.Synced` is true only when live processing is ready and the committed
  Wallet tip exactly matches the observed chain tip by height and hash.
- Keep `Info.SyncedTo` equal to the committed Wallet tip.
- Remove `IsRecoveryMode` and `RecoveryProgress`; do not replace them with
  fields exposing internal live-sync or historical-rescan state.
- Targeted rescans must not alter live readiness, another Wallet's readiness,
  or `SyncedTo`.
- Preserve the existing small-gap operation policy even while `Synced` is
  false.
- Keep the result valid before and after Task 423 changes the chain-event
  owner. Task 428 owns public integration assertions for exported fields.
- Exclude RPC, deprecated-wallet behavior, polling helpers, and rescan or
  notification protocol redesign.

## Suggested fix

Derive the existing public fields from a consistent view of the committed
Wallet tip and the private observed live tip. The internal snapshot mechanism
is not prescribed.

## Verification

- Cover readiness with exact match, height mismatch, hash mismatch, source
  failure, and delivery-not-ready cases.
- Prove a small gap reports `Synced=false` without changing operation
  admission.
- Start, complete, and fail a targeted rescan and prove live status and a
  second Wallet remain independent.
- Run Task 433's existing synchronization scenario unchanged on kvdb, SQLite,
  and PostgreSQL.

## Definition of done

`Wallet.Info` reports exact live synchronization through `Synced` and
`SyncedTo`, removes unsupported recovery projections, and remains independent
of targeted rescans and backend ownership changes.

[t334]: 334-shared-chain-state.md
[t428]: ../../tasks/428-controller-sync-info-itests.md
[t433]: 433-wallet-sync-test-boundary.md
