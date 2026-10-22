# Task 419 — measure address-read performance

Component: **wallet AddressManager benchmarks**.

Type: **Benchmark**.

Dependencies: **[Task 471][t471]**.

## Rationale

ListAddresses and GetAddressInfo measure list and point reads. Preserve their
existing legacy/new kvdb comparisons and reuse selected cases for SQL, rather
than replacing those baselines or reproducing coupled growth variables.

## Details

- Follow the [scoped benchmark exception][backend-rule], reusing
  [Task 471][t471]'s minimal SQL setup.
- Put new SQL benchmarks in `wallet/bench`, using public `wallet` APIs and
  Task 471's common Docker environment for all four implementations.
- Measure each method separately with one representative existing workload.
  Match account identity/count, actual total addresses, script types, funded
  state, request filters, expected results, and the point-selection rule with
  its selected baseline. A few explicit sizes may vary address count alone.
- Seed and validate once per case, then repeatedly call the selected method
  on the same active Wallet. Do not reopen or reset between reads.
- Validate actual seeded state, complete list semantics, and the point result
  outside timing. Keep setup and cleanup outside timing and fail on unexpected
  operation errors.
- Use the same timing boundaries for a fair comparison in both suites.
  Exclude Docker/environment setup, database startup/readiness, initial
  connections, schema/data creation, and teardown; normal API database I/O
  remains measured. Keep any selected baseline alignment local.
- Capture separate CPU and memory `pprof` profiles under Task 471's profile
  rules; exclude preparation and teardown and keep profiling out of timing runs.
- Add one short file per method under btcwallet's `docs/developer/benchmark/`:
  method/commit SHA(s), methodology/Docker environment, a data table, and one
  unprofiled ns/op, B/op, allocs/op results table. Include
  legacy kvdb, modern kvdb, PostgreSQL, and SQLite, with percentage changes
  against legacy kvdb. Reuse the existing suite's kvdb measurements;
  record profile locations and capture/analysis commands. No CPU time/RSS
  reporting or new profiling/reporting framework is required.
- Exclude cold/warm modes, broad legacy cleanup, imports, mutations, independent
  balance measurements, concurrency, tuning, and production changes.

## Suggested fix

Reuse selected `BenchmarkListAddressesAPI` and `BenchmarkGetAddressInfoAPI`
workloads with focused SQL loops, preserving both existing kvdb measurements.
Keep any alignment local to the selected cases and use explicit deterministic
data, without a workload framework or migration of unrelated benchmarks.

## Verification

- Run each selected SQL method/case and validate equivalent public results
  and timed work against separately obtained existing kvdb measurements.
- Confirm only the selected operation is timed, each result names its workload,
  and every fixture is cleaned up.
- Verify environment and database setup are untimed in both suites and the
  method records include the data table, one results table with percentage
  changes, and usable CPU/memory profiles captured separately in Docker.

## Definition of done

Both public address reads have focused SQL measurements matched to selected
existing workloads, with short records under `docs/developer/benchmark/`,
CPU/memory profiles, preserved kvdb comparisons, and no production changes.

[backend-rule]: ../AGENTS.md#scope-rules
[t471]: 471-three-backend-list-unspent-benchmark.md
