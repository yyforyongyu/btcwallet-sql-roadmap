# Task 476 — read legacy SQL walletdb sources

Component: **Legacy migration source adapter**.

Type: **Migration**.

Dependencies: **[Task 374][t374]**.

## Rationale

Existing conversion reads a bbolt source. Legacy SQLite and PostgreSQL
walletdb wallets expose the same logical buckets but cannot be handed to the
maintained Manager, and the SQL driver's Copy operation is unavailable.

## Details

- Read a consistent, preserved legacy SQLite or PostgreSQL walletdb source
  into the existing normalized extraction result.
- Reuse Tasks 373, 473, and 374; do not duplicate their fact mapping.
- Keep source access and extraction separate from destination import and lnd
  database cutover. Exclude deprecated etcd.
- Reject an unsupported source shape before claiming a complete snapshot.

## Suggested fix

Use existing walletdb readers over a stable backend-specific source view and
feed the common migration extraction boundary.

## Verification

For each SQL source, verify source preservation, a consistent complete
snapshot, and all supported keys, counters, imports, transactions, UTXOs,
leases, birthday, and sync facts. Task 485 owns conversion and reopen.

## Definition of done

Both legacy SQL source formats yield the complete normalized snapshot without
mutating the user's original database.

[t374]: 374-extract-legacy-transaction-sync-state.md
