# Task 375 — import one legacy snapshot into SQLite

Component: **SQLite migration importer**.

Type: **Migration**.

Dependencies: **[Task 329][t329], [Task 374][t374], and [Task 411][t411]**.

## Rationale

Ordinary Store methods commit wallet, account, address, transaction, UTXO,
lease, and sync mutations independently. Calling them in sequence can expose a
partial converted wallet. The complete normalized snapshot needs one
SQLite-only mutation owner that builds on Task 329's final method-specific SQL
genesis transaction shape without widening the general Store API.

## Details

- Accept only the complete internal normalized snapshot produced by
  Task 374. Do not receive a source path, source Store, legacy manager, or
  plaintext secret.
- Support one fresh SQLite destination. PostgreSQL conversion requires a later
  consumer-backed task and is not implied by Task 333.
- Require Task 340's exact SQLite schema/network identity and prove that the
  destination contains no wallet state before beginning the import. A definite
  failure may leave only the preexisting validated empty schema and identity;
  it cannot leave any migrated wallet row or child artifact.
- Add one migration-specific SQLite write transaction that inserts the wallet,
  ciphertext secrets, sync state, scopes, accounts, account secrets, immutable
  discovery modes, checked cursors, addresses, derivation paths, blocks,
  transactions, statuses, labels, UTXOs, spend/replacement edges, and active
  leases.
- Before the first write, apply Task 411's terminal-parent admission predicate
  to every normalized parent edge in the snapshot. Refuse the complete import
  with the exact child and terminal parent when any child would be inadmissible
  through ordinary direct or batch ingestion; migration must not bypass the
  live transaction contract.
- Reuse Task 329's final dialect-neutral genesis stages or lower-level
  method-specific insertion helpers where their contract matches. Do not route
  the import through repeated `db.Store` calls, add a generic Store callback, or
  turn the Task 329 genesis operation into a generic importer.
- Resolve all destination row IDs inside the transaction from stable normalized
  identities. Never persist a kvdb bucket key, synthetic imported account
  number, or adapter row identity as public SQL identity.
- On every definite validation, insertion, or commit failure, roll back the
  complete wallet import. Return the SQL runtime's ambiguous-commit result
  unchanged for Task 376; do not retry, delete, or claim rollback when the
  outcome is unknown.
- Never open or mutate the source. Do not verify public conversion equivalence,
  publish a Wallet, select a cutover, or expose a public API.
- Exclude PostgreSQL, generic manifests, replay markers, generic locks,
  cross-process fencing, ordinary Store API expansion, and automatic cutover.

## Suggested fix

Add a SQLite-only migration import method with its own narrow ops interface and
one `runtime.Write` callback. Feed it ciphertext and public facts from the
normalized snapshot, bind every child row inside that transaction, and use the
same definite-versus-ambiguous commit classification as other SQL writes.

## Verification

- A complete representative snapshot imports into a fresh SQLite database and
  a raw reopen observes every identity, account, script, graph, lease, block,
  and sync fact.
- Stage-level fault injection at wallet, secret, sync, scope, account,
  account-secret, discovery-mode, cursor, address, derivation, block,
  transaction, credit/debit, UTXO, replacement, and lease insertion proves a
  definite failure leaves no wallet state.
- A non-fresh, wrong-network, foreign, dirty, or mismatched destination fails
  before import mutation.
- A snapshot containing a child of a terminal transaction is refused with the
  exact child/parent identity and leaves no imported wallet state; equivalent
  direct, batch, scan, and migration vectors produce the same Task 411 result.
- An injected ambiguous commit is returned to the caller without retry,
  cleanup, publication, or an unsupported success/failure claim.
- Dependency spies prove the importer never accesses a legacy source and never
  routes through a sequence of ordinary Store methods.

## Definition of done

- One migration-specific SQLite transaction imports every supported normalized
  wallet fact or none of them after a definite result.
- The importer consumes only ciphertext/public facts, never mutates the source,
  and leaves no partial wallet in a fresh destination.
- Imported transaction graphs satisfy Task 411's terminal-parent predicate and
  cannot enter SQL through a migration-only bypass.
- Ambiguous commit state is preserved for the final orchestration task.
- The PR adds no PostgreSQL converter, generic Store import API, manifest,
  replay, lock, public orchestration, or cutover behavior.

[t329]: 329-atomic-wallet-account-creation.md
[t374]: 374-extract-legacy-transaction-sync-state.md
[t411]: 411-terminal-parent-rejection.md
