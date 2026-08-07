# Task 349 — expose idempotent transaction invalidation

Component: **TxWriter and TxPublisher**.

Dependencies: **[Task 334][t334], [Task 339][t339], and [Task 340][t340], plus
an accepted ADR 0006 amendment making normalized inputs authoritative for
dependency traversal**.

## Rationale

SQL retains invalidated transaction history internally, but modern callers
cannot invalidate an unmined conflict branch. Dependency discovery also lacks a
normalized input relation and otherwise must decode raw transactions.

## Details

- Add a narrow public branch-invalidation operation for SQL.
- kvdb public calls return unsupported; deprecated removal behavior remains.
- Retain transaction identity and history; do not physically delete modern
  branches.
- Persist every input, including external inputs, in the same transaction as
  `raw_tx`.
- Backfill existing raw transactions deterministically and verify normalized
  inputs against decoded wire inputs before switching readers. Recompute each
  decoded transaction ID and require exact equality with every independent
  persisted `tx_hash` value. If preflight finds a hash mismatch or an active
  child of a terminal parent, reject the migration with exact diagnostics
  rather than silently legitimizing or repairing the graph.
- Amend ADR 0006 before making normalized inputs authoritative for dependency
  traversal; it currently names raw transaction decoding as the source.
- Update `tx_invalidation_flows.md` and `utxo_data_model.md` at the same cutover
  so no developer contract still claims UTXO-edge-only traversal.
- Make replay idempotent without clearing a later valid spender.
- Reject every ingestion whose normalized input references a terminal parent
  transaction before writing the child or any credit/debit effects.
- Route every transaction status or graph mutation through the same
  operation-specific admission, including insert, `UpdateTx`, `ApplyTxBatch`,
  `ApplyScanBatch`, `DeleteTx`, invalidation, wallet rewind, and rollback. Do
  not add a generic Store write lock.
- Make `DeleteTx` remove normalized input rows atomically, using a verified
  cascade or explicit child-first deletion.
- Let [Task 334][t334]'s database-wide rollback coordinator acquire graph admission for
  every affected wallet in stable wallet-ID order before mutation. A rollback
  cannot take only one wallet's gate or invert the per-wallet acquisition order.
- Prevent publication from crossing invalidation between durable admission and
  network send. Use the same operation-specific admission or a durable pre-send
  status that invalidation can atomically reject; do not hold a database
  transaction across the network call.
- Make descendant traversal cycle-safe and enforce a documented depth or work
  limit. Exceeding the limit fails the whole invalidation atomically.
- Treat an invalidation commit error as ambiguous. Re-read the exact durable
  operation marker and member/edge result, returning success, definite failure,
  or a wallet-owned indeterminate error.

## Potential fix

Add normalized `transaction_inputs` plus indexed parent-outpoint lookup and a
forward backfill migration. Retain an invalidation marker containing the exact
member/edge result, use a recursive SQL query for descendants, and acquire the
same wallet mutation exclusion used by transaction insertion, rollback, and
publisher admission. Validate all parent statuses within that admission. Add
cycle detection and a bounded traversal budget before updating any status.
Use the same marker to reconcile an ambiguous commit; do not add a generic
transaction journal.

## Done when

Root and descendants become terminal while unrelated siblings and later valid
spenders survive; new descendants of terminal parents are rejected, and an
invalidation-versus-publication barrier test permits only one valid ordering.
Every named graph writer participates, database-wide rollback acquires wallets
deterministically, and delete removes its normalized edges atomically. Backfill
refuses an already-invalid graph, ambiguous commit reconciles exactly, and
replay remains correct after restart on SQLite and PostgreSQL. Corrupt raw-tx
versus persisted-hash fixtures fail atomic preflight on both dialects.

[t334]: 334-shared-chain-state.md
[t339]: 339-transaction-visibility.md
[t340]: 340-durable-database-identity.md
