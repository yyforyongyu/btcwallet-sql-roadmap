# Task 477 — import a legacy snapshot into PostgreSQL

Component: **PostgreSQL migration importer**.

Type: **Migration**.

Dependencies: **[Task 374][t374], [Task 478][t478],
[Task 480][t480], and [Task 481][t481]**.

## Rationale

Task 375 imports into native SQLite only. A legacy PostgreSQL walletdb source
needs a native PostgreSQL destination without a second extraction pipeline.

## Details

- Atomically import the common normalized snapshot into a wallet-empty native
  PostgreSQL Manager database.
- Preserve supported wallet facts, reject wrong-network or occupied
  destinations, and report ambiguous commits without retry or overwrite.
- Keep legacy source access and Manager.Create orchestration out of scope.

## Suggested fix

Reuse Task 375's snapshot model and destination invariants in a PostgreSQL
transaction, without a source-specific importer.

## Verification

Import a representative full snapshot, reopen it, verify signing and state
fidelity, and inject definite and ambiguous commit failures. Preserve the
legacy source; Task 485 owns the public conversion path.

## Definition of done

A complete normalized wallet snapshot imports into PostgreSQL atomically or
reports failure without falsely claiming rollback.

[t374]: 374-extract-legacy-transaction-sync-state.md

[t478]: 478-retain-spend-confirmation-leases.md
[t480]: ../archive/tasks/480-own-raw-imports-in-one-wallet.md
[t481]: ../archive/tasks/481-persist-imported-account-address-schema.md
