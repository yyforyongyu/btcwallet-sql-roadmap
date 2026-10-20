# Task 410 — coordinate transaction graph mutations

Component: **Wallet-owned SQL transaction mutation ordering**.

Type: **Bug fix**.

Dependencies: **[Task 334][t334]**.

## Rationale

Descendant invalidation reads a graph snapshot before changing statuses and
spend links. If maintained Wallet ingestion can insert a child after that
snapshot without joining the same ordering boundary, TxReader can retain an
active child of an invalidated parent and UtxoManager can retain its spend.
A transaction around each writer does not alone prove that this interleaving
is excluded. At `sql-wallet` revision
`4236325cbb609881272f488e413b90ab4aa07fc6`, PostgreSQL `runtime.Write` still
uses `BeginTx(ctx, nil)`. Controller handlers and syncer batch writes can run
concurrently. This is source-level evidence, not a reproduced race; this task
owns validation of existing locks, predicates, triggers, and driver behavior
before selecting a correction. No separate ADR is required.

Candidate schedule under PostgreSQL read committed: active unmined A spends
wallet coin F and creates A:0; a confirmed winner W spends F. W's ingestion
snapshots A's descendants before child B exists. Concurrent ingestion commits
B spending A:0 and independent wallet coin Q while A is still active. W then
marks A replaced and processes only its earlier descendant IDs. Check whether
B remains active, its credit visible, and Q spent after both operations settle.
Use maintained confirmed-conflict ingestion or public `DeleteUnconfirmedTx`,
not a new public invalidation API, to establish reachability.

SQLite already uses `_txlock=immediate`, acquiring writer ownership before
these reads and excluding this particular two-writer schedule. Preserve that
protection; PostgreSQL does not justify matching application machinery on
SQLite.

## Details

- Validate the candidate ingestion-versus-invalidation interleaving through
  maintained Wallet behavior and preserve one consistent public result in
  either ordering. If current protections already exclude it, record that
  evidence and supersede the unnecessary fix. If child admission wins first,
  invalidation must include it; if the terminal transition wins first,
  admission must respect the retained terminal parent.
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

- Use barriers to reproduce the public failure on PostgreSQL before the fix.
  Exercise both orderings afterward on both SQL backends, preserving SQLite
  writer exclusion rather than requiring a pre-fix SQLite failure.
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
