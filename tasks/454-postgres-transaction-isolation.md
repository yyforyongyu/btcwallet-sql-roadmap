# Task 454 — make PostgreSQL transaction isolation explicit

Component: **PostgreSQL Store transaction runtime**.

Type: **Bug fix**.

Dependencies: **None**.

## Rationale

Normal PostgreSQL Store writes currently begin without transaction options, so
their isolation inherits the server, database, role, or DSN default. lnd uses
the same PostgreSQL DSN for multiple stores and selects serializable isolation
for its own transactions. A native btcwallet Store may share that database but
must not silently change behavior when its co-tenant or operator changes a
default, nor pay serializable contention costs for operations whose invariants
need only read-committed execution.

## Details

- Define a private, driver-neutral isolation policy for normal PostgreSQL Store
  transactions and require every transactional entry point to select it
  explicitly.
- Use read committed only where constraints, exact compare-and-set predicates,
  or acquired row locks preserve the operation's complete invariant. Permit
  serializable isolation only for an operation that documents a predicate or
  multi-row invariant that cannot be protected more narrowly.
- Do not add repeatable-read or another profile without a current operation
  that proves it needs that exact snapshot contract.
- Keep isolation selection in btcwallet code. Do not change PostgreSQL server,
  database, role, DSN, or lnd configuration, and do not expose a public tuning
  knob that lets callers weaken Store correctness.
- Keep serialization and deadlock classification stable. Do not add a generic
  write retry: an operation using serializable isolation must separately prove
  that its complete callback is replay-safe or return its existing classified
  conflict or operation-specific reconciliation result.
- Preserve SQLite behavior and backend-neutral Store results. The native pgx
  cutover in [Task 350][t350] must retain the selected policy rather than fall
  back to pgx defaults or globally strengthen isolation.
- Exclude connection-pool sizing, query optimization, schema placement, lnd
  production changes, and persistent benchmark code. [Task 416][t416] excludes
  concurrent writers; a separately accepted benchmark task owns PostgreSQL
  contention, abort-rate, retry, and co-tenant load measurements.

## Suggested fix

Add the smallest internal transaction-policy value accepted by the PostgreSQL
write runtime, map it to explicit driver transaction options, and update each
normal PostgreSQL write entry point to name its policy. Keep the policy
driver-neutral so Task 350 can map the same choice to pgx without changing
callers or Store behavior.

## Verification

- Run focused PostgreSQL runtime tests that capture every begin option and fail
  when an entry point leaves isolation unspecified.
- Override the session or role default in a real PostgreSQL fixture and prove
  representative ordinary writes still execute at their explicit
  read-committed level. If the audit admits a stronger level, prove its owning
  operation executes at that exact level too.
- Use two Store instances to exercise one representative conflicting operation
  per admitted policy and prove no lost update, write skew, partial effect, or
  silently retried callback.
- Inject serialization and deadlock failures and verify stable classification,
  context cancellation, and that no callback is silently retried.

## Definition of done

- Every normal PostgreSQL Store transaction selects an explicit reviewed
  isolation policy independent of database defaults and lnd co-tenants.
- Each admitted level has a concrete correctness reason and concurrent
  PostgreSQL coverage; no stronger level is used as a generic safety blanket.
- SQLite and public Wallet behavior remain unchanged. The driver-neutral
  policy contract is complete without the later native pgx cutover.
- The PR adds no public isolation setting, global write retry, lnd change,
  connection tuning, or benchmark suite.

[t350]: 350-native-pgx-postgres.md
[t416]: 416-database-benchmark-fixtures.md
