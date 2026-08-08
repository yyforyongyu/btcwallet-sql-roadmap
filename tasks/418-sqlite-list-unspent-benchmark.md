# Task 418 — measure SQLite ListUnspent performance

Component: **wallet UtxoManager benchmarks**.

Dependencies: **[Task 416][t416]**.

## Rationale

`Wallet.ListUnspent` loads Store UTXO rows, enriches them into public values,
applies the account-name filter, and sorts the result. The existing benchmark
uses the legacy walletdb fixture, so it cannot show how the SQLite query and
wallet-side conversion scale with both the live set and relevant transaction
history required by [Issue #1251][issue1251].

## Details

- Add exactly `BenchmarkSQLiteListUnspent`.
- Run `Cold` and `Warm` sub-benchmarks for
  `Utxo-U100-H100`, `Utxo-U100-H10000`,
  `Utxo-U10000-H10000`, and `Utxo-U10000-H100000`.
- Query all accounts with `Account: ""`, `MinConfs: 0`, and
  `MaxConfs: math.MaxInt32`. Do not add account-filter, lease, concurrency, or
  mutation variants in this task.
- A cold iteration reloads its snapshot outside the timer and times exactly
  one call to `Wallet.ListUnspent`. A warm sub-benchmark opens once, performs
  one untimed priming call, resets the timer, and times one public call per
  iteration.
- Stop the timer before checking result length, ordering, outpoint identity,
  and values.
- Call `b.ReportAllocs`. Keep fixture construction, reload, validation, and
  teardown outside the timer.
- Do not change UTXO queries, indexes, sorting, allocation strategy, cache
  behavior, or wallet semantics. A measured problem becomes a separate task.

## Suggested fix

Add the SQL cases beside the existing UtxoManager benchmarks while preserving
their names and legacy comparison purpose. Instantiate each stable snapshot
through the reusable fixture and use one helper to enforce identical cold and
warm boundaries.

## Verification

- Run:

  ```text
  go test ./wallet -run '^$' \
    -bench '^BenchmarkSQLiteListUnspent$' -benchmem \
    -benchtime=3s -count=10 | tee listunspent.txt
  benchstat listunspent.txt
  ```

- Compare revisions with
  `benchstat listunspent-base.txt listunspent-change.txt`.
- Confirm all eight profile/cache combinations report `ns/op`, `B/op`, and
  `allocs/op` and return exactly the named `U` live outputs.
- Confirm no database creation, population, or reload falls inside the timed
  public call.

## Definition of done

- SQLite `ListUnspent` results scale across both named live-set and history
  dimensions.
- Cold and warm measurements time exactly one public read.
- Allocations and repeatable benchstat comparison are documented.
- Correctness checks prove count, identity, value, and sort order outside the
  timer.
- No tuning, mutation benchmark, P1 work, or PostgreSQL lane is included.
- The PR contains only the ListUnspent measurement outcome.

[t416]: 416-sqlite-benchmark-fixtures.md
[issue1251]: https://github.com/btcsuite/btcwallet/issues/1251
