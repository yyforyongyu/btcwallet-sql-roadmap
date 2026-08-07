# Task 333 — add PostgreSQL Manager and harness support

Component: **Manager and integration-test harness**.

Dependencies: **None**. The PostgreSQL Store and the unified Manager API are
already present.

## Rationale

The PostgreSQL Store exists, but `NewManager` and the daemon harness currently
support only kvdb and SQLite. Enabling PostgreSQL early lets every new public
component itest run against both SQL dialects and expose backend differences
while those tests are being written.

## Details

- Keep the single `NewManager(ctx, ManagerConfig)` constructor.
- Add one `DBBackendPostgres` backend case; do not add another config type.
- Reuse `ManagerConfig.DataSource` as the DSN and `MaxConnections` as the pool
  bound.
- Share the existing SQL Manager create/load implementation between SQLite and
  PostgreSQL. Keep only backend opening and configuration dialect-specific; do
  not copy the Manager logic into a PostgreSQL implementation.
- Preserve one database per Manager with wallets separated by `wallet_id`.
- Teach `bwtest` to provision one PostgreSQL server per itest process and an
  isolated database per subtest. Close Managers before dropping their database,
  and stop the server during suite teardown.
- Make `db=postgres` run the same registered test cases as kvdb and SQLite. Do
  not copy or fork component test cases by backend.
- Replace the file-header assertion only for PostgreSQL with a real connection
  or schema probe; retain the existing kvdb and SQLite artifact checks.
- Keep this task to backend selection, lifecycle, and harness plumbing. It does
  not add new itest cases or change wallet behavior merely because PostgreSQL
  exposes an existing bug.
- Defer durable database/network identity to [Task 340][t340], shared multiwallet chain
  coordination to [Task 334][t334], and duplicate-create semantics to [Task 330][t330]. Bugs
  found by the PostgreSQL run should be fixed by their owning task or recorded
  as a new task rather than absorbed here.
- Do not expand the default CI matrix without an explicit cost decision.

## Potential fix

Extract the dialect-neutral parts of `sqliteManagerBackend` into one SQL
`managerBackend`. Keep small SQLite and PostgreSQL open functions that construct
their Store and return that shared backend. Then teach `bwtest` to provision an
isolated PostgreSQL database and pass its DSN through the existing
`ManagerConfig`.

## Done when

`make itest chain=btcd db=postgres icase=manager` and the existing Controller
case run against a real, isolated PostgreSQL database through `NewManager`.
Closing and reopening a Manager against the same DSN exercises the normal load
path. The same test source remains selectable for kvdb, SQLite, and PostgreSQL,
and teardown leaves no Manager, connection pool, database, or container behind.
No new component scenario, multiwallet chain coordinator, database identity
scheme, or default CI matrix expansion lands in this task.

[t330]: 330-duplicate-wallet-create.md
[t334]: 334-shared-chain-state.md
[t340]: 340-durable-database-identity.md
