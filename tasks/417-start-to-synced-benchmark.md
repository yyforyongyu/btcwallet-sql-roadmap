# Task 417 — measure start-to-synced performance

Component: **Manager lifecycle and Syncer benchmarks**.

Type: **Benchmark**.

Dependencies: **[Task 331][t331], [Task 471][t471], and [Task 455][t455]**.

## Rationale

Existing Controller benchmarks mix startup with live-chain RPC, mining, and
profiling and sleep-based completion polling. Preserve their legacy/new kvdb
comparison and measure the matching public startup-to-synced workflow on SQL;
equal block counts alone do not make the two measurements comparable.

## Details

- Follow the [scoped benchmark exception][backend-rule], reusing
  [Task 471][t471]'s minimal SQL setup.
- Select a representative existing empty- or populated-chain startup case.
  Match Wallet/account counts, initial durable state, chain source and daemon
  behavior, block history and match pattern, and completion observation with
  the selected baseline. Do not compare live-RPC timings with in-process-chain
  timings merely because block counts agree. A few explicit block counts may
  vary that dimension alone.
- Before each sample, prepare the same starting Wallet state and construct a
  fresh Manager. Never restart a terminal Manager or Wallet. Keep this local
  setup in the startup benchmark rather than adding reset modes to Task 471.
- Time Manager.Start through observation of the exact expected synchronized
  tip using public Wallet information; align the selected legacy startup
  boundary and completion condition to equivalent work. Include completion
  observation; exclude construction, seeding, validation, and Stop.
- Prepare deterministic chain data before timing. Normal database and matched
  chain-source I/O remain measured; mining, random data generation, profiling,
  and artificial completion sleeps do not.
- For a fair comparison in both suites, exclude Docker/environment setup,
  database startup/readiness, initial connections, schema/data creation, and
  teardown. Database server startup is separate from the measured Manager
  startup-to-synced workflow.
- Validate actual initial and final public state outside timing and keep any
  selected baseline alignment local.
- Add a short file for this method under btcwallet's `docs/benchmark/`: method
  and wallet commit SHA(s), methodology/timing boundary, a data/parameters
  table, and a time/allocation results table for legacy kvdb, modern kvdb,
  PostgreSQL, and SQLite. Reuse existing kvdb results; no reporting framework
  is required.
- Add no cold/warm variants, cache controls, concurrency matrix, profiling,
  tuning, or production diagnostics.

## Suggested fix

Reuse a selected existing startup workload and chain fixture for the focused
SQL loop, making only necessary local baseline adjustments for comparable
timing. Reuse public synchronization observation and untimed setup/teardown,
without a second lifecycle framework or cleanup of every startup benchmark.

## Verification

- Run selected SQL cases and require the exact final tip and Wallet-visible
  state. Validate equivalent initial state and startup/completion boundaries
  against the separately obtained existing kvdb results.
- Confirm each sample starts from equivalent state, measures the same public
  workflow, and leaves no live Manager, Wallet, or backend resources.
- Verify environment and database setup are untimed in both suites and the
  concise method record includes both the data and four-result tables.

## Definition of done

One SQL start-to-synced measurement matches a selected existing startup
workload and its timing/completion boundary, with a short `docs/benchmark/`
record and no production changes or replacement of kvdb comparisons.

[backend-rule]: ../AGENTS.md#scope-rules
[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t471]: 471-three-backend-list-unspent-benchmark.md
[t455]: ../archive/tasks/455-exact-wallet-sync-status.md
