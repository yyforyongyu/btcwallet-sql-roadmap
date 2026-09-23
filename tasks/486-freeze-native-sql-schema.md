# Task 486 — freeze native SQL schema for fresh Wallets

Component: **SQLite and PostgreSQL schema initialization**.

Type: **Migration**.

Dependencies: **[Task 391][t391], [Task 478][t478],
[Task 480][t480], and [Task 481][t481]**.

## Rationale

Before `sql-wallet` merges into `master`, the planned account, import, and
lease storage shapes must be final. The current runtime embeds versioned
upgrade/downgrade migrations, which would violate the release's fresh-schema
boundary.

## Details

- Initialize a new SQLite or PostgreSQL database with the final Wallet schema,
  preserving current identity, constraints, indexes, and triggers.
- Remove versioned `.up.sql` and `.down.sql` files and the runtime rollback
  path from the merged tree. Update SQL subsystem guidance accordingly.
- Reopen a current database without changing its schema. Reject incompatible
  pre-merge native-SQL databases explicitly instead of silently upgrading.
- Exclude legacy walletdb data conversion, which has separate Phase 3 owners.

## Suggested fix

Reuse the accepted dialect-specific DDL as fresh-database initialization and
retire the versioned migration runner after dependent schema work lands.

## Verification

Create and reopen fresh Wallets on both SQL dialects; verify the final schema
and refusal of an incompatible earlier schema. Check that no upgrade/downgrade
files or rollback runner remain. Replace obsolete rollback tests with the
fresh-initialization contract.

## Definition of done

Both SQL backends initialize the final schema directly, and the merged tree
contains no versioned upgrade/downgrade files or implicit upgrade path.

[t391]: 391-path-bound-watch-only-xpub-accounts.md
[t478]: 478-retain-spend-confirmation-leases.md
[t480]: 480-own-raw-imports-in-one-wallet.md
[t481]: 481-persist-imported-account-address-schema.md
