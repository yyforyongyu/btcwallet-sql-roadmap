# Task 420 — measure transaction-read performance

Component: **wallet TxReader benchmarks**.

Type: **Benchmark**.

Dependencies: **[Task 471][t471]**.

## Rationale

GetTx and ListTxns measure point and range reads over transaction history.
Preserve their existing legacy/new kvdb comparisons and reuse selected cases
for SQL. Existing growth labels alone do not establish matching history or
transaction complexity.

## Details

- Follow the [scoped benchmark exception][backend-rule], reusing
  [Task 471][t471]'s minimal SQL setup.
- Measure each method separately with one representative existing workload.
  Match actual persisted history count, account/address state, input/output
  shape, confirmed/unmined mix, point selection, and ListTxns filters and
  expected content. Include the whole seeded history in the list request.
  A few explicit sizes may vary history count alone, keeping complexity fixed.
- Seed and validate once per case, then repeatedly call the selected method
  on the same active Wallet. Do not reopen or reset between reads.
- Validate actual state, the point result, and complete list outside timing.
  Keep setup and cleanup outside timing and fail on unexpected operation errors.
- Use the same timing boundaries for a fair comparison in both suites.
  Exclude Docker/environment setup, database startup/readiness, initial
  connections, schema/data creation, and teardown; normal API database I/O
  remains measured. Keep any selected baseline alignment local.
- Measure Wallet CPU time/op and workload RSS using Task 471's measurement
  scope and setup exclusions; keep PostgreSQL server resources separate.
- Add one short file per method under btcwallet's `docs/developer/benchmark/`:
  method/commit SHA(s), methodology/data, and two results tables. First show
  elapsed time/op, CPU time/op, and Wallet RSS; then B/op and allocs/op. Include
  legacy kvdb, modern kvdb, PostgreSQL, and SQLite, with percentage changes
  against legacy kvdb. Reuse the existing suite's kvdb measurements;
  no reporting framework is required.
- Exclude cold/warm modes, broad legacy cleanup, labels, writes,
  invalidation, concurrency, tuning, and production changes.

## Suggested fix

Reuse selected serial `BenchmarkGetTxAPI` and `BenchmarkListTxnsAPI` workloads
with focused SQL loops, preserving both existing kvdb measurements. Keep any
alignment local to the selected cases and their actual data and requests;
concurrent variants, profiles, and unrelated benchmarks remain out of scope.

## Verification

- Run each selected SQL method/case and validate equivalent public results
  and timed work against separately obtained existing kvdb measurements.
- Confirm only the selected operation is timed, each result names its workload,
  and every fixture is cleaned up.
- Verify environment and database setup are untimed in both suites and the
  concise method records include both results tables and percentage changes.

## Definition of done

Both public transaction reads have focused SQL measurements matched to
selected existing workloads, with short records under
`docs/developer/benchmark/`, preserved kvdb comparisons, and no production
changes.

[backend-rule]: ../AGENTS.md#scope-rules
[t471]: 471-three-backend-list-unspent-benchmark.md
