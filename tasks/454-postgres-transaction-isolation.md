# Task 454 — make PostgreSQL transaction isolation explicit

Component: **PostgreSQL Store transaction runtime**.

Type: **Bug fix**.

Dependencies: **[Task 410][t410]**.

## Rationale

Normal PostgreSQL Store writes currently begin without transaction options, so
their isolation inherits the server, database, role, or DSN default. lnd uses
the same PostgreSQL DSN for multiple stores and selects serializable isolation
for its own transactions. A native btcwallet Store may share that database but
must not silently change behavior when its co-tenant or operator changes a
default, nor pay serializable contention costs for operations whose invariants
need only read-committed execution.

## Details

- Set explicit driver transaction options in the existing PostgreSQL write
  runtime. One documented default suffices where it preserves current
  invariants; require an operation-specific override only for a demonstrated
  need, not a new policy abstraction or per-entry-point selector.
- Use read committed only where constraints, exact compare-and-set predicates,
  or acquired row locks preserve the operation's complete invariant. Permit
  serializable isolation only for an operation that documents a predicate or
  multi-row invariant that cannot be protected more narrowly.
- Preserve [Task 410][t410]'s proved graph-ordering contract when making
  isolation explicit. Task 410 alone
  owns the graph correction, any graph-specific isolation choice and its
  justification, and the corresponding regression. This task must neither
  weaken nor foreclose that choice; it does not select or prove the graph fix.
  Task 410 must merge first so this policy preserves its graph correction
  and any required isolation level.
- Do not add repeatable-read or another profile without a current operation
  that proves it needs that exact snapshot contract.
- Keep isolation selection in btcwallet code. Do not change PostgreSQL server,
  database, role, DSN, or lnd configuration, and do not expose a public tuning
  knob that lets callers weaken Store correctness.
- Keep serialization and deadlock classification stable. Do not add a generic
  write retry: an operation using serializable isolation must separately prove
  that its complete callback is replay-safe or return its existing classified
  conflict or operation-specific reconciliation result.
- Preserve SQLite behavior and backend-neutral Store results. Use the driver
  already in place: if [Task 350][t350] lands first, set native pgx options;
  otherwise its later cutover must preserve this explicit policy. Neither
  task depends on the other's completion.
- Exclude connection-pool sizing, query optimization, schema placement, lnd
  production changes, and persistent benchmark code. [Task 471][t471] excludes
  concurrent writers; a separately accepted benchmark task owns PostgreSQL
  contention, abort-rate, retry, and co-tenant load measurements.

## Suggested fix

Set the justified explicit isolation level at the existing PostgreSQL write
boundary, using database/sql or native pgx as present. Add a stronger operation
option only where a current invariant requires it; no new driver-neutral
policy type is required.

## Verification

- Prove the shared PostgreSQL write boundary supplies explicit isolation;
  cover a separate entry point only if it bypasses that boundary.
- Override the session or role default in a real PostgreSQL fixture and prove
  representative ordinary writes still execute at their explicit
  read-committed level. If the audit admits a stronger level, prove its owning
  operation executes at that exact level too.
- Use two Store instances to exercise one representative conflicting operation
  per admitted policy and prove no lost update, write skew, partial effect, or
  silently retried callback. Task 410 owns the graph interleaving regression;
  reuse it to verify that the explicit policy preserves the correction,
  without adding duplicate cases.
- Inject serialization and deadlock failures and verify stable classification,
  context cancellation, and that no callback is silently retried.

## Definition of done

- Every normal PostgreSQL Store transaction selects an explicit reviewed
  isolation policy independent of database defaults and lnd co-tenants.
- Each admitted level has a concrete correctness reason and concurrent
  PostgreSQL coverage; no stronger level is used as a generic safety blanket.
- SQLite and public Wallet behavior remain unchanged. Explicit isolation does
  not require the native pgx cutover to land first.
- The PR adds no public isolation setting, global write retry, lnd change,
  connection tuning, or benchmark suite.

[t350]: 350-native-pgx-postgres.md
[t471]: 471-three-backend-list-unspent-benchmark.md
[t410]: 410-transaction-graph-admission.md
