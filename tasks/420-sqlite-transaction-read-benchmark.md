# Task 420 — measure SQLite transaction-read performance

Component: **wallet TxReader benchmarks**.

Dependencies: **[Task 416][t416]**.

## Rationale

`GetTx` and `ListTxns` are the P0 public TxReader paths over transaction detail
and range queries. Existing benchmarks use legacy walletdb state and vary
transaction pool and I/O dimensions with growth functions. A shared immutable
SQLite history fixture gives [Issue #1251][issue1251] stable, named, directly
comparable measurements of both reads without combining their timers.

## Details

- Add exactly `BenchmarkSQLiteTransactionReads` with separate `GetTx` and
  `ListTxns` sub-benchmark trees.
- For both APIs, run `Cold` and `Warm` against `Tx-T100`, `Tx-T1000`, and
  `Tx-T10000`.
- `GetTx` always requests the deterministic median confirmed transaction hash
  from the manifest. `ListTxns` always requests heights `0` through `-1`, the
  maintained all-history boundary.
- A cold iteration reloads outside the timer and times exactly one public API
  call. A warm sub-benchmark opens once, performs one untimed call to the same
  API, resets the timer, and times one call per iteration.
- Stop before checking the target transaction detail or the complete list's
  count, ordering, confirmation state, inputs, outputs, and block membership.
- Call `b.ReportAllocs` in every leaf benchmark.
- Exclude concurrent reads, different input/output shapes, labels, writes,
  invalidation, tuning, PostgreSQL, and P1 paths.

## Suggested fix

Add SQL sub-benchmarks beside `tx_reader_benchmark_test.go`. Reuse one profile
manifest for the `GetTx` target and `ListTxns` expected history, but retain one
timer per public method and leave existing legacy comparison benchmarks
unchanged.

## Verification

- Run:

  ```text
  go test ./wallet -run '^$' \
    -bench '^BenchmarkSQLiteTransactionReads$' -benchmem \
    -benchtime=3s -count=10 | tee transaction-reads.txt
  benchstat transaction-reads.txt
  ```

- Compare revisions with
  `benchstat transaction-reads-base.txt transaction-reads-change.txt`.
- Confirm all twelve API/cache/profile leaves report `ns/op`, `B/op`, and
  `allocs/op`.
- Confirm no fixture construction, reload, or correctness traversal runs
  within either API timer.

## Definition of done

- Both P0 TxReader APIs run against the same stable SQLite history profiles.
- Each API has an independent cold and warm timing boundary.
- Correctness checks cover confirmed/unmined state and complete public detail
  outside the timer.
- Allocation reporting and repeatable benchstat comparison are documented.
- No tuning, write path, invalidation work, P1 benchmark, or PostgreSQL lane is
  included.
- The PR contains only the coherent TxReader measurement outcome.

[t416]: 416-sqlite-benchmark-fixtures.md
[issue1251]: https://github.com/btcsuite/btcwallet/issues/1251
