# Task 483 — preserve labels on chain replay

Component: **Wallet scan ingestion and TxStore batch**.

Type: **Bug fix**.

Dependencies: **None**.

## Rationale

When `ApplySyncScanBatch` reobserves a confirmed transaction already labeled
`external`, the empty chain label conflicts with the stored label. The batch
returns `tx already exists` and Wallet sync cannot advance.

## Details

- Treat an absent chain label as no replacement for the same transaction.
- Keep transaction effects and sync-tip advancement atomic on SQLite and
  PostgreSQL, including concurrent label changes.
- Preserve direct `CreateTx` conflicting-label rejection. Exclude a new event
  system, public API, and schema migration.

## Suggested fix

Reuse scan batch/idempotent replay handling and resolve the effective label
within the same transactional boundary.

## Verification

Replay a labeled confirmed transaction and check label, credits/spends, and
advanced tip; retain direct conflicting-label tests. Task 470 reruns lnd's
unchanged btcd `last_unused_addr` case. Local PoC:
`a5984443e2d62c64196207c9c307cc4925415609`.

## Definition of done

A repeated chain observation cannot stall sync or erase a stored user label.
