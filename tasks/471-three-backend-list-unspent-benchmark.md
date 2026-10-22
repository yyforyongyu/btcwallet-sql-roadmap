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
  comparison suite and reuse selected workloads and kvdb measurements.
- Construct focused SQL Wallets normally through Manager, reusing real-backend
  setup and teardown. Add only the minimal SQL setup needed here;
  [417][t417], [419][t419], and [420][t420] reuse it and own their data and
  baseline alignment. Their dependency is on shared infrastructure, not this
  task's performance result. Task 417 owns per-sample startup work.

### Matching workloads

- Select one representative existing ListUnspent workload and fixed request.
  A few explicit sizes may vary live UTXO count alone; record supporting
  history without another scaling dimension. Do not recreate the old matrix.
- Use a small data-only definition or an equally simple explicit description
  shared by both suites; matching names or configuration counts alone do not
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
  Wallet. Validate count, order, outpoints, and values outside timing and fail
  on operation errors.
- Use equivalent workloads and timing boundaries for a fair comparison across
  all four implementations. Exclude Docker/environment provisioning, database
  startup and readiness, initial connections, schema/data creation, correctness
  checks, and teardown. Measure only ListUnspent, including its normal DB I/O.
- Measure elapsed time/op, Wallet-process CPU time/op (user plus system), and
  steady-state RSS in MiB during the workload. Record prepared-Wallet idle RSS
  and the sampling method; exclude setup and other cases' retained state.
  Process-lifetime peak RSS does not establish workload-only memory usage.
- Collect CPU/RSS separately if sampling would distort timing. Label
  PostgreSQL server CPU/RAM separately from Wallet-process measurements.
  Obtain missing kvdb resource measurements in the existing suite, never by
  adding kvdb execution to the focused SQL group.
- Exclude production fixes, tuning, harness retirement, broad cleanup, leases,
  mutation, concurrency, daemon matrices, cold/warm modes, cache controls,
  snapshot/reset machinery, generic runners, and custom statistics services.

### Benchmark record

Keep one short file per method under btcwallet's `docs/developer/benchmark/`,
for example `docs/developer/benchmark/list_unspent.md`:

- The method and wallet commit SHA(s) used for the measurements.
- Brief methodology, timing boundary, data, and resource-measurement scope.
  Use the same workload for all four implementations.
- Two results tables for legacy kvdb, modern kvdb, PostgreSQL, and SQLite:
  elapsed time/op, CPU time/op, and Wallet RSS first; B/op and allocs/op second.
  Show percentage changes against legacy kvdb for each metric. Reuse existing
  kvdb measurements; note retired implementations or unmeasured values.
- B/op and allocs/op describe allocation traffic, not resident memory. Never
  infer CPU consumption from elapsed time or RSS from allocation metrics.

No shared export format, automated result joining, or reporting framework is
required. SQL execution remains independent of stored baseline results.

Suggested record:

```markdown
# ListUnspent

Wallet commit: <full SHA>

Methodology: <measured operation and timing boundary; setup is untimed>

Data: <accounts, addresses, history, live UTXOs, confirmation mix, filters>

Resources: <CPU/RSS sampling method and interval; prepared-Wallet idle RSS>

| Implementation | Time/op (µs; Δ) | CPU/op (µs; Δ) | Wallet RSS (MiB; Δ) |
| --- | ---: | ---: | ---: |
| Legacy kvdb | ... (baseline) | ... (baseline) | ... (baseline) |
| Modern kvdb | ... (↑/↓...%) | ... (↑/↓...%) | ... (↑/↓...%) |
| PostgreSQL | ... (↑/↓...%) | ... (↑/↓...%) | ... (↑/↓...%) |
| SQLite | ... (↑/↓...%) | ... (↑/↓...%) | ... (↑/↓...%) |

| Implementation | B/op | Δ | allocs/op | Δ |
| --- | ---: | ---: | ---: | ---: |
| Legacy kvdb | ... | Baseline | ... | Baseline |
| Modern kvdb | ... | ↑/↓...% | ... | ↑/↓...% |
| PostgreSQL | ... | ↑/↓...% | ... | ↑/↓...% |
| SQLite | ... | ↑/↓...% | ... | ↑/↓...% |
```

If results use different commits, add a commit SHA column to the results table.
Δ is the signed percentage change from legacy kvdb, not a confidence interval.
Label any PostgreSQL server resource measurements separately.

## Suggested fix

Select an existing case, describe and validate its actual workload, and reuse
its kvdb measurements with an independent Manager-based SQL loop. Exercise
minimal shared setup through ListUnspent itself, rather than prebuilding later
consumers or adding a setup-only demonstration project.

## Verification

- Run the selected SQL cases and validate matching public results outside
  timing. Obtain both kvdb measurements separately from the existing suite;
  check equivalent state, requests, and timing boundaries. Reuse contract
  tests rather than repeat their assertion matrices.
- Verify Docker and DB setup are untimed in both suites, with complete
  setup/failure cleanup and independent SQL execution after kvdb retirement.
- Verify CPU/RSS measurements cover the workload window and identify the
  prepared-Wallet baseline; do not substitute allocation data or lifetime peaks.
- Check the method record contains commit SHA(s), methodology/data, and both
  results tables with measurements and changes against legacy kvdb.

## Definition of done

One focused SQL ListUnspent measurement exercises minimal reusable setup and
provides a short method record under `docs/developer/benchmark/` with fair
speed, CPU, RSS, and allocation comparisons. Existing kvdb benchmarks remain,
without production changes or a reporting framework.

[source]: https://github.com/btcsuite/btcwallet/blob/4236325cbb609881272f488e413b90ab4aa07fc6/wallet/utxo_manager_benchmark_test.go
[backend-rule]: ../AGENTS.md#scope-rules
[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t416]: ../archive/tasks/416-database-benchmark-fixtures.md
[t418]: ../archive/tasks/418-list-unspent-benchmark.md
[t417]: 417-start-to-synced-benchmark.md
[t419]: 419-address-read-benchmark.md
[t420]: 420-transaction-read-benchmark.md
