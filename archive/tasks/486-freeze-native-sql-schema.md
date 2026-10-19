# Task 486 — Consolidate initial SQL schema before master merge

> Superseded on 2026-10-19: the accepted decision in [issue #56][issue56]
> preserves separate SQL files and the existing migration loader. Tasks 391,
> 478, 480, and 481 own the schema changes; fresh creation and unchanged reopen
> remain pre-merge checks. No replacement task or completion credit is added.

Component: **SQLite and PostgreSQL schema initialization**.

Type: **Migration**.

Dependencies: **[Task 391][t391], [Task 478][t478],
[Task 480][t480], and [Task 481][t481]**.

## Rationale

Before `sql-wallet` merges into `master`, the planned account, import, and
lease storage shapes must be final. Consolidate their development-era schema
changes into a clean initial baseline for SQLite and PostgreSQL for the initial
release.
The existing migration loader can initialize this baseline.

## Details

- Consolidate development-era schema changes after Tasks 391, 478, 480, and
  481 into a clean initial migration for each SQL dialect.
- Retain the existing migration loader. Fresh SQLite and PostgreSQL databases
  must receive the final Wallet schema, preserving its identity, tables,
  columns, constraints, indexes, and triggers.
- Update affected SQL-generation references, generated output, tests, and SQL
  subsystem guidance to use the consolidated initial baseline.
- Reopen a database created from that baseline without changing its schema.
- Exclude migration-framework removal, experimental-schema compatibility
  detection or rejection, and legacy walletdb conversion; legacy conversion
  has separate Phase 3 owners.

## Suggested fix

Fold the accepted dialect-specific schema changes into the initial migration
sources and reuse the existing loader. Refresh SQL-generation references and
tests where consolidation invalidates them.

## Verification

- Create fresh Wallets on SQLite and PostgreSQL using the clean initial
  migration and existing loader.
- Reuse and update existing focused schema tests to verify the final tables,
  columns, constraints, indexes, and triggers, including the dependent schema
  changes. Reopen those databases and verify their schemas remain unchanged.
- Run affected SQL generation and tests against the consolidated sources;
  remove or update assertions tied to obsolete development migration steps.

## Definition of done

Both SQL backends use a clean initial migration through the existing loader.
Focused tests verify the final schema's identity, tables, columns, constraints,
indexes, and triggers, fresh creation, and unchanged reopen. Affected
SQL-generation references, generated output, tests, and guidance match the
consolidated baseline.

[t391]: 391-path-bound-watch-only-xpub-accounts.md
[t478]: ../../tasks/478-retain-spend-confirmation-leases.md
[t480]: 480-own-raw-imports-in-one-wallet.md
[t481]: 481-persist-imported-account-address-schema.md

[issue56]: https://github.com/yyforyongyu/btcwallet-sql-roadmap/issues/56
