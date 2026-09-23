# Task 485 — convert legacy SQL wallets through Manager

Component: **Manager.Create legacy conversion selection**.

Type: **Migration**.

Dependencies: **[Task 376][t376], [Task 476][t476], and
[Task 477][t477]**.

## Rationale

Task 376 exposes bbolt-to-SQLite conversion only. Existing lnd SQLite and
PostgreSQL walletdb users need a native-SQL conversion path before old SQL
walletdb startup can be removed.

## Details

- Reuse Manager.Create conversion lifecycle for legacy SQLite walletdb to
  native SQLite and legacy PostgreSQL walletdb to native PostgreSQL.
- Preserve the source; reject occupied destinations and unrepresentable
  custody or lease state rather than dropping it.
- Keep lnd deployment/database cutover separate; exclude etcd and arbitrary
  source/destination combinations.

## Suggested fix

Select Task 476's source adapter and Task 375 or 477's importer within the
existing Task 376 creation and publication flow.

## Verification

For each source/destination pair, convert representative signing and imported
wallet state, reopen and sign, compare supported state, and confirm source
preservation and refusal without destination overwrite.

## Definition of done

Both legacy SQL walletdb formats have a verified native-SQL Manager.Create
conversion route without a parallel public converter.

[t376]: 376-orchestrate-legacy-conversion.md
[t476]: 476-read-legacy-sql-walletdb-source.md
[t477]: 477-import-legacy-snapshot-postgres.md
