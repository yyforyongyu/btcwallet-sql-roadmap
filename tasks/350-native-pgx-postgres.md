# Task 350 — use native pgx for PostgreSQL storage

Component: **PostgreSQL Store**.

Dependencies: **none**.

## Rationale

The PostgreSQL adapter still crosses `database/sql`, while its natural driver
is `pgx/v5`. Shared DB helpers should not expose either backend's nullable or
transaction types.

## Details

- Introduce backend-neutral nullable/domain values in `wallet/internal/db`.
- Keep SQLite on `database/sql` internally.
- Generate PostgreSQL sqlc code for `pgx/v5` and use `pgxpool.Pool`/`pgx.Tx`.
- Replace the shared runtime contract that exposes `*sql.DB` and `*sql.Tx` with
  a backend-neutral transaction runner or a pgx-native counterpart. Preserve
  health state, error classification, retry accounting, and ambiguous-commit
  reporting.
- Migrate PostgreSQL migrations and integration helpers to native pgx.
- Remove `lib/pq` only after no runtime or generated use remains.

## Potential fix

Land shared nullable types and the backend-neutral runtime seam first, adapt
SQLite and kvdb without changing behavior, then switch PostgreSQL sqlc
generation, Store transactions, migrations, and tests in separate buildable
commits.

## Done when

No PostgreSQL runtime path uses `database/sql` or `lib/pq`, shared packages
expose no driver types, runtime health and ambiguous-commit semantics remain
equivalent, and PostgreSQL Store conformance remains green.
