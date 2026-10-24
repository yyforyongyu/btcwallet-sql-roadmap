# Task 490 — measure GetUtxo performance

Component: **wallet UtxoManager benchmarks**.

Type: **Benchmark**.

Dependencies: **[Task 471][t471]**.

## Rationale

[BenchmarkGetUtxoAPI][source] compares legacy and maintained Wallet kvdb point
reads. Task 471 measures ListUnspent, leaving the SQL single-output lookup
without focused performance measurements.

## Details

- Follow the [scoped benchmark exception][backend-rule] and Task 471's shared
  setup. Put new SQL benchmarks in `wallet/bench`, using public Wallet APIs.
  Keep kvdb execution in the existing suite; the focused suite runs only
  SQLite/PostgreSQL and remains independent of kvdb retirement.
- Select one representative existing workload and a known unspent outpoint.
  Match account/scope, address/script types, persisted transaction shape,
  confirmation state, live UTXOs, and point selection with both kvdb baselines.
  A few explicit sizes may vary history count alone; keep other inputs fixed.
- Seed once, then repeatedly call GetUtxo on the same active Wallet. Keep
  fixture-state and result-content validation in unit/integration tests;
  fail on setup or operation errors.
- Use matched timing boundaries and Task 471's common Docker environment for
  a fair comparison. Exclude environment setup, database startup/readiness,
  initial connections, schema/data creation, and teardown; normal API DB I/O
  remains measured.
- Start with the smallest dataset and one selected case. Build the code, run
  relevant tests, and smoke-run that case. Present code and results for user
  approval before full datasets, all cases, or full profile captures.
- Follow Task 471's separate CPU/memory pprof capture rules and report format
  in `docs/developer/benchmark/get_utxo.md`. Include the data table and one
  unprofiled ns/op, B/op, allocs/op results table for legacy kvdb, modern kvdb,
  PostgreSQL, and SQLite, with percentage changes against legacy kvdb.
- Exclude other API measurements, production changes, tuning, concurrency,
  broad harness cleanup, and a new reporting framework.

## Suggested fix

Reuse one existing GetUtxo workload and its kvdb measurements with a focused
SQL loop over Task 471's setup. Keep any baseline alignment local to that case.

## Verification

- Build, run relevant tests, and smoke-run the smallest case before seeking
  explicit user approval for full measurements and profiles.
- After approval, run the selected cases and retain separate CPU/memory
  profiles. Keep setup untimed and record matching workload/timing settings.
- Reproduce the method record's exact benchmark and profile commands in the
  documented Docker environment; retain profile locations and wallet SHAs.

## Definition of done

GetUtxo has focused SQL measurements, matching existing kvdb baselines, and a
short method record with the agreed data/results tables and separate profiles.
The full run follows explicit user approval; production code is unchanged.

[source]: https://github.com/btcsuite/btcwallet/blob/18a4a11aa7cb66c3f9def3f8df9a01217f261519/wallet/utxo_manager_benchmark_test.go#L18
[backend-rule]: ../AGENTS.md#scope-rules
[t471]: 471-three-backend-list-unspent-benchmark.md
