# Task 333 — add PostgreSQL Manager and harness support

Component: **Manager and integration-test harness**.

Type: **Gap fix**.

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
- Defer durable database/network identity to [Task 340][t340] and
  duplicate-create semantics to [Task 330][t330].
- Treat [Task 334][t334] as the shared-chain design owner. Runtime
  chain-source delivery, canonical frontier mutation, and multi-Wallet
  rollback belong to [Task 370][t370], [Task 371][t371], and
  [Task 372][t372], while live SQL Manager activation belongs to
  [Task 423][t423]. Bugs found by the PostgreSQL run should be fixed by their
  owning task or recorded as a new task rather than absorbed here.
- Do not expand the default CI matrix without an explicit cost decision.

## Suggested fix

Extract the dialect-neutral parts of `sqliteManagerBackend` into one SQL
`managerBackend`. Keep small SQLite and PostgreSQL open functions that construct
their Store and return that shared backend. Then teach `bwtest` to provision an
isolated PostgreSQL database and pass its DSN through the existing
`ManagerConfig`.

## Verification

`make itest chain=btcd db=postgres icase=manager` and the existing Controller
case run against a real, isolated PostgreSQL database through `NewManager`.
Closing and reopening a Manager against the same DSN exercises the normal load
path. The same test source remains selectable for kvdb, SQLite, and PostgreSQL,
and teardown leaves no Manager, connection pool, database, or container behind.
No new component scenario, multiwallet chain coordinator, database identity
scheme, or default CI matrix expansion lands in this task.

## Definition of done

- PostgreSQL is selectable through the existing Manager constructor and the
  shared component-itest source with isolated lifecycle management.
- SQLite and kvdb retain their existing selection and artifact checks.
- The PR contains only backend-opening and harness plumbing, not defects owned
  by [Task 330][t330], [Task 340][t340], or shared-chain work owned by
  [Task 334][t334], Tasks 370–372, and [Task 423][t423].

[t330]: ../../tasks/330-duplicate-wallet-create.md
[t334]: ../../tasks/334-shared-chain-state.md
[t340]: ../../tasks/340-durable-database-identity.md
[t370]: ../../tasks/370-chain-source-delivery.md
[t371]: 371-canonical-chain-frontier.md
[t372]: ../../tasks/372-multi-wallet-rollback.md
[t423]: ../../tasks/423-manager-chain-source-cutover.md
