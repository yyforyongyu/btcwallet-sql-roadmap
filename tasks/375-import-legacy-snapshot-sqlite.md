# Task 375 — import one legacy snapshot into SQLite

Component: **SQLite migration importer**.

Type: **Migration**.

Dependencies: **[Task 340][t340], [Task 374][t374], [Task 478][t478],
[Task 480][t480], and [Task 481][t481]**.

## Rationale

Ordinary Store calls commit wallet facts independently. Using them in sequence
could expose a partially converted wallet, so the complete validated snapshot
needs one migration-specific SQLite transaction.

## Details

- Consume [Task 373][t373]'s wallet/account/ciphertext output,
  [Task 473][t473]'s address/script/ciphertext output, and Task 374's
  transaction/sync output from the same validated source snapshot. No stage
  alone contains the complete import. Accept no source handle or plaintext
  in this boundary.
- Import into a wallet-empty SQLite database with the validated network
  identity. Schema and empty-Manager bootstrap metadata may exist, but no
  prior wallet, account, transaction, or UTXO state may exist. Initial
  conversion does not add a wallet alongside existing wallets.
- Check that precondition inside the import transaction. Preserve compatible
  bootstrap chain facts and reject a conflict rather than overwrite them.
- Insert every supported wallet, account, address, transaction, UTXO, lease,
  block, and sync fact in one atomic migration operation. Validate all foreign
  identities and current SQL invariants inside this boundary.
- Resolve SQL row IDs from normalized identities; never persist legacy bucket,
  synthetic-account, or adapter identifiers.
- A definite failure leaves no imported wallet state. Return an ambiguous
  commit for Task 376 to expose as `ErrIndeterminateCommit`; do not retry,
  reconcile by readback, or delete potentially committed state.
- Exclude PostgreSQL, ordinary Store API expansion, public orchestration,
  source access, and automatic cutover.

## Suggested fix

Use one SQLite migration transaction with a narrow private ops boundary. It may
reuse lower-level insertion helpers, but need not depend on the ordinary wallet
genesis or future transaction-graph implementation.

## Verification

- Import a representative complete snapshot and directly verify every
  normalized identity and state after reopen.
- Inject failures across the major insertion groups and prove no wallet state
  remains after a definite result.
- Accept normal empty-Manager bootstrap metadata. Reject an occupied,
  wrong-network, dirty, or conflicting destination before partial import.
- Preserve an injected ambiguous commit for Task 376 without retry, readback
  reconciliation, or cleanup of potentially committed state.

## Definition of done

One SQLite transaction imports every supported normalized wallet fact or none
of them after a definite result, independently of ordinary genesis and
speculative transaction-graph tasks.

[t340]: ../archive/tasks/340-durable-database-identity.md
[t373]: 373-extract-legacy-account-address-state.md
[t374]: 374-extract-legacy-transaction-sync-state.md
[t473]: 473-extract-legacy-address-script-state.md

[t478]: 478-retain-spend-confirmation-leases.md
[t480]: ../archive/tasks/480-own-raw-imports-in-one-wallet.md
[t481]: ../archive/tasks/481-persist-imported-account-address-schema.md
