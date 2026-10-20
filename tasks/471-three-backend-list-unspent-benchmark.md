# Task 471 — measure SQL ListUnspent with kvdb baselines

Component: **wallet UtxoManager benchmarks and shared benchmark setup**.

Type: **Benchmark**.

Dependencies: **[Task 331][t331]**.

## Rationale

At `sql-wallet` revision `4236325cbb609881272f488e413b90ab4aa07fc6`,
[BenchmarkListUnspentAPI][source] compares legacy and maintained Wallet kvdb
calls. Preserve those comparisons and reuse selected workloads and results
for SQL measurement. The existing fixture varies accounts, addresses, and
transactions together; its labels alone do not establish comparable state.

On 2026-09-19, this task combines [416][t416] and [418][t418]. The first real
measurement exercises the shared setup directly, removing the need for a
separate setup-only example and smoke matrix.

## Details

- Follow the [scoped benchmark exception][backend-rule]. Preserve the existing
  comparison suite; own only selected-case alignment, selection, validation,
  and baseline export alongside the first useful SQL ListUnspent measurement.
- Construct focused SQL Wallets normally through Manager, reusing real-backend
  setup and teardown. Add only the minimal setup and reporting needed here;
  [417][t417], [419][t419], and [420][t420] reuse it and own their data and
  baseline alignment. Their dependency is on shared infrastructure, not this
  task's performance result. Task 417 owns per-sample startup work.

### Matching workloads

- Select one representative existing ListUnspent workload and fixed request.
  A few explicit sizes may vary live UTXO count alone; record supporting
  history without another scaling dimension. Do not recreate the old matrix.
- Use a small data-only definition or an equally simple explicit description
  shared by both suites. Give each case a stable identity and distinguish
  changes to its definition; matching names or configuration counts do not
  establish equivalent work.
- Match account identity and selection, total addresses and script types,
  owned/foreign inputs, persisted history count and transaction shape,
  confirmed/unconfirmed mix, live UTXOs and values, chain tip, lock/sync state,
  request filters, expected results, and timed operation/completion boundary.
  Fix seeds, keys, scripts, times, hashes, and values where they affect work.
- Validate actual logical state outside timing; backend seeding and internal
  IDs may differ. Existing builders count addresses per account, make four
  transaction passes, cap batches by available addresses, and jitter values
  with wall-clock time. Align only the selected case where needed, rather
  than rewriting helpers or migrating unrelated benchmarks.
- Check observable equivalence of selected legacy adapters. Do not compare
  different semantics, such as force-next allocation versus unused reuse;
  mark a genuinely unmatched baseline unavailable.
- Seed and validate once, then repeatedly call ListUnspent on the same active
  Wallet. Keep setup, correctness checks, and teardown outside timing, validate
  count, order, outpoints, and values, and fail on operation errors.

### Reporting contract

- This task owns the shared reporting contract used by 417/419/420. Each suite
  exports its own measurements; the reporting step executes no Wallet calls.
  Join by method and workload identity, showing legacy kvdb, maintained kvdb,
  SQLite, and PostgreSQL columns where available.
- Report time, allocated bytes, allocations, exact dimensions, and direct
  differences against comparable legacy results. Never multiply ratios from
  different workloads to estimate old-to-SQL performance.
- Record result source revision, workload definition, toolchain, machine, and
  relevant database/environment settings. Label historical results as recorded
  evidence. Never silently join changed workloads or incompatible environments.
- SQL execution requires no baseline artifact. SQL-only absolute reporting is
  valid; a requested comparison must identify missing or unusable baselines
  instead of claiming four implementations were compared. Rerun or locally
  align unusable selected baselines in the existing suite when needed.
- Use ordinary Go benchmark output and established tools such as benchstat
  with explicit implementation labels and minimal export/normalization glue.
  Existing benchmark code does not prove usable result files already exist.
- Require explicit review of unexplained comparable regressions as Phase 2B
  validation, without an arbitrary universal percentage limit. These selected
  methods establish no performance conclusion for every Wallet operation.
- Exclude production fixes, tuning, harness retirement, broad cleanup, leases,
  mutation, concurrency, daemon matrices, cold/warm modes, cache controls,
  snapshot/reset machinery, generic runners, and custom statistics services.

## Suggested fix

Select an existing case, describe and validate its actual workload, and reuse
its kvdb measurements with an independent Manager-based SQL loop. Exercise
minimal shared setup and reporting through ListUnspent itself, rather than
prebuilding later consumers or adding a setup-only demonstration project.

## Verification

- Run the selected SQL cases and validate matching public results outside
  timing. Obtain both kvdb measurements separately from the existing suite;
  check actual baseline state, requests, and measurement boundaries before
  joining. Reuse contract tests rather than repeat their assertion matrices.
- Verify operation-only timing, complete setup/failure cleanup, and the
  execution separation and independent removability in the scoped exception.
- Produce the first report with provenance and direct matched comparisons
  where available. Verify baseline-absent SQL execution/reporting, explicit
  missing columns, and rejection of incompatible result combinations.

## Definition of done

One focused SQL ListUnspent measurement exercises minimal reusable setup and
the first matched comparison report, preserving existing kvdb comparisons.
Evidence identifies missing baselines and unexplained regressions without
inventing measurements, production changes, or a replacement framework.

[source]: https://github.com/btcsuite/btcwallet/blob/4236325cbb609881272f488e413b90ab4aa07fc6/wallet/utxo_manager_benchmark_test.go
[backend-rule]: ../AGENTS.md#scope-rules
[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t416]: ../archive/tasks/416-database-benchmark-fixtures.md
[t418]: ../archive/tasks/418-list-unspent-benchmark.md
[t417]: 417-start-to-synced-benchmark.md
[t419]: 419-address-read-benchmark.md
[t420]: 420-transaction-read-benchmark.md
