# Task 350 — use native pgx for PostgreSQL storage

Component: **PostgreSQL Store**.

Dependencies: **None**.

## Rationale

Normal PostgreSQL Store queries and transactions still cross
`database/sql`, while their native driver and error model are pgx. The shared
runtime must not force SQLite's transaction types onto PostgreSQL.

## Details

- Generate PostgreSQL sqlc queries for `pgx/v5` and use
  `pgxpool.Pool`/`pgx.Tx` for normal Store reads and writes.
- Keep SQLite on `database/sql`.
- Refactor shared nullable/domain conversions and runtime transaction hooks
  only as far as needed to prevent either backend's driver types from escaping
  the shared `wallet/internal/db` contract.
- Preserve health state, retry classification and accounting, fatal-error
  poisoning, and ambiguous-commit reporting.
- Keep the existing migration library and test-administration connection on
  `database/sql` when that external tool boundary requires it. They are not
  normal Store query/transaction paths.
- Do not replace the migration framework, rewrite migration SQL, or require
  removal of `lib/pq` while the selected migration driver still owns it.

## Suggested fix

Make the shared runtime transaction primitive backend-parametric or add the
smallest pgx-native counterpart with the same hook contract, switch the
PostgreSQL sqlc package and Store to it, and centralize pgx nullable conversion
at the PostgreSQL adapter boundary. Leave SQLite and the migration tool
unchanged.

## Verification

- PostgreSQL Store conformance and runtime tests execute normal reads and
  writes through `pgxpool` and `pgx.Tx` with no `database/sql` Store handle.
- Retry, health, fatal-error, and ambiguous-commit vectors remain equivalent
  to the existing runtime contract.
- Shared DB interfaces and domain values expose no pgx or `database/sql`
  types; SQLite conformance remains green.
- A source audit permits `database/sql` only in SQLite and the documented
  migration/test-administration tool boundaries.

## Definition of done

- Normal PostgreSQL Store queries and transactions use native pgx while
  SQLite retains `database/sql`.
- Shared runtime and domain contracts are driver-neutral and preserve all
  current failure semantics.
- The PR contains no migration-framework replacement or unrelated storage
  redesign.

The shared runtime seam stays in Task 350: by itself it changes no externally
useful behavior and exists only to enable this cutover, so splitting it would
create a preparatory task that is not independently useful.
