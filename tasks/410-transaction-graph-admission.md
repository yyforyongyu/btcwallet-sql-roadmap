# Task 410 — coordinate transaction graph mutations

Component: **Wallet-owned SQL transaction mutation ordering**.

Type: **Bug fix**.

Dependencies: **[Task 334][t334] and [Task 424][t424]**.

## Rationale

Descendant invalidation reads a graph snapshot before changing statuses and
spend links. If maintained Wallet ingestion can insert a child after that
snapshot without joining the same ordering boundary, TxReader can retain an
active child of an invalidated parent and UtxoManager can retain its spend.
A transaction around each writer does not alone prove that this interleaving
is excluded. At `sql-wallet` revision `2ae306dea8c5`, PostgreSQL
`runtime.Write` uses `BeginTx(ctx, nil)`: under read committed, invalidation
can read the active graph before concurrent ingestion adds a child and then
update only its earlier IDs. Controller handlers and syncer batch writes are
concurrent entry points. Task 424 validates this path and existing protection
before selecting the correction.

## Details

- Reproduce the accepted ingestion-versus-invalidation interleaving through
  maintained Wallet behavior and preserve one consistent public result in
  either ordering. If current protections already exclude it, record that
  evidence and supersede the unnecessary fix.
- Coordinate only graph mutations participating in the demonstrated race:
  direct/batch ingestion, state changes, invalidation, deletion, rewind, or
  rollback where reachable. Do not route unrelated writes through a new gate.
- Keep transaction status, descendants, and spend ownership coherent. Preserve
  independent work in unrelated Wallets and avoid deadlock for database-wide
  rollback under [Task 372][t372].
- Reuse existing transactions, conditional writes, or local synchronization
  where sufficient. No graph coordinator, admission-handle type, or per-Wallet
  serialization service is required by this task.
- Keep terminal-parent admission in Task 411. Existing exact-outpoint spend
  predicates and raw graph reads remain; Tasks 409 and 413 are superseded
  representation/duplicate work, not prerequisites.
- Exclude generic write locks, network calls inside database transactions,
  publication policy, new public APIs, and cross-process fencing.

## Suggested fix

Protect the smallest existing read/mutate boundary that admits the reproduced
race. Choose the mechanism from the accepted evidence, not from a desired
final coordinator architecture.

## Verification

- Use barriers to reproduce the public failure before the fix and exercise
  both orderings afterward on SQLite and PostgreSQL.
- Check TxReader status and UtxoManager spendability after the operations
  settle; no mixed graph result may remain.
- Verify unrelated Wallet operations still proceed and rollback cannot
  deadlock with an admitted mutation.
- Test the behavioral invariant rather than the existence of a lock, handle,
  new type, or serialized call count.

## Definition of done

The demonstrated concurrent mutation defect is closed at the Wallet-owned
boundary with focused regressions and no unnecessary coordination layer.

[t334]: ../archive/tasks/334-shared-chain-state.md
[t372]: 372-multi-wallet-rollback.md
[t424]: 424-transaction-utxo-integrity-adr.md
