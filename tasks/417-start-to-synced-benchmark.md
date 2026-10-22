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
- Put new SQL benchmarks in `wallet/bench`, using public `wallet` APIs and
  Task 471's common Docker environment for all four implementations.
- Select a representative existing empty- or populated-chain startup case.
  Match Wallet/account counts, initial durable state, chain source and daemon
  behavior, block history and match pattern, and completion observation with
  the selected baseline. Do not compare live-RPC timings with in-process-chain
  timings merely because block counts agree. A few explicit block counts may
  vary that dimension alone.
- Before each sample, prepare the same starting Wallet state and construct a
  fresh Manager. Never restart a terminal Manager or Wallet. Keep this local
  setup in the startup benchmark rather than adding reset modes to Task 471.
- Time Manager.Start through observation of the configured synchronized tip
  using public Wallet information; align the selected legacy startup
  boundary and completion condition to equivalent work. Include completion
  observation only to end the sample; exclude construction, seeding, and Stop.
- Prepare deterministic chain data before timing. Normal database and matched
  chain-source I/O remain measured; mining, random data generation, profiling,
  and artificial completion sleeps do not.
- For a fair comparison in both suites, exclude Docker/environment setup,
  database startup/readiness, initial connections, schema/data creation, and
  teardown. Database server startup is separate from the measured Manager
  startup-to-synced workflow.
- Leave fixture-state and result-content validation to unit/integration
  tests. Fail on setup or operation errors; keep baseline alignment local.
- Capture separate CPU and memory `pprof` profiles under Task 471's profile
  rules; exclude preparation and teardown and keep profiling out of timing runs.
- Add a short method file under btcwallet's `docs/developer/benchmark/`:
  start with the method title and succinct description of how it is run, then
  `### Benchmark Results`, `### Data`, and `### Commands`, in that order, as in
  [Task 471's example][t471]. Include method/commit SHA(s),
  methodology/Docker environment, a data table, and one
  unprofiled ns/op, B/op, allocs/op results table. Include
  legacy kvdb, modern kvdb, PostgreSQL, and SQLite, with percentage changes
  against legacy kvdb. Reuse the existing suite's kvdb measurements;
  record profile locations and exact benchmark/capture/analysis commands,
  including Docker invocation and working directories. No CPU time/RSS
  reporting or new profiling/reporting framework is required.
- Add no cold/warm variants, cache controls, concurrency matrix,
  tuning, or production diagnostics.

## Suggested fix

Reuse a selected existing startup workload and chain fixture for the focused
SQL loop, making only necessary local baseline adjustments for comparable
timing. Reuse public synchronization observation and untimed setup/teardown,
without a second lifecycle framework or cleanup of every startup benchmark.

## Verification

- Run selected SQL cases and record performance against separately obtained
  existing kvdb measurements using the documented workload.
- Confirm each sample measures the documented startup/completion boundary
  and leaves no live Manager, Wallet, or backend resources.
- Verify environment and database setup are untimed in both suites and the
  method record includes the data table, one results table with percentage
  changes, and usable CPU/memory profiles captured separately in Docker.
- Reproduce the recorded benchmark and profile runs using `Commands`.

## Definition of done

One SQL start-to-synced measurement matches a selected existing startup
workload and its timing/completion boundary, with a short method record under
`docs/developer/benchmark/`, CPU/memory profiles, and no production changes or
kvdb replacement.

[backend-rule]: ../AGENTS.md#scope-rules
[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t471]: 471-three-backend-list-unspent-benchmark.md
[t455]: ../archive/tasks/455-exact-wallet-sync-status.md
