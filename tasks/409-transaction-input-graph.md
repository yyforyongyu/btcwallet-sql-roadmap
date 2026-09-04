# Task 409 — normalize transaction input edges

Component: **SQL schema and TxStore ingestion**.

Type: **Gap fix**.

Dependencies: **[Task 340][t340] and [Task 424][t424]**.

## Rationale

Add one wallet-scoped normalized row for every
transaction input, including external inputs, with input position and previous
outpoint. The new persisted input position and previous-output index must each
cover the wire range `0..MaxUint32` in both SQL dialects. Task 424 supplies the
accepted normalized-input contract.

## Details

Persist it atomically with `raw_tx` in direct and batch ingestion.
Backfill existing transactions by decoding raw bytes; recompute and compare
each txid; reject hash mismatch, an input position or previous-output index
outside `0..MaxUint32`, duplicate input positions/outpoints, malformed raw
bytes, or an active child of an already terminal parent with exact diagnostics
before cutover. Add dialect-equivalent range constraints to both new index
columns. Switch invalidation, rollback descendant reads, and deletion/cascade
behavior to normalized edges and update maintained graph documentation in the
same migration.

Exclusions: public invalidation, terminal-parent admission for concurrent
new writes, mutation admission, UTXO spend-pointer constraints, publication,
and raw transaction removal.

## Suggested fix

Add a wallet-scoped `transaction_inputs` table with a
referenceable unique input identity, a constrained wire-range position, and an
indexed previous outpoint whose index has the same wire-range constraint.
Populate it in every ingestion transaction, then use a preflighted forward
migration to switch graph readers.

## Verification

SQLite/PostgreSQL empty and populated migrations produce
identical input rows and reject adjacent out-of-range input positions and
previous-output indexes before cutover. Boundary fixtures accept zero and
`MaxUint32`, including the valid coinbase previous-output index. Corrupt
hash/raw/edge fixtures fail atomically; direct, batch, scan, delete, rollback,
and invalidation fixtures no longer decode raw transactions for graph
traversal.

## Definition of done

Normalized inputs are the sole authoritative graph
edge source on both dialects, all ingestion/deletion paths maintain them, and
their input positions and previous-output indexes are constrained to
`0..MaxUint32`; ADR/data-model text matches the cutover.

[t340]: ../archive/tasks/340-durable-database-identity.md
[t424]: 424-transaction-utxo-integrity-adr.md
