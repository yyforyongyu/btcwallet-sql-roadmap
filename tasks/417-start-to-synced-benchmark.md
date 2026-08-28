# Task 417 — measure start-to-synced performance

Component: **Manager lifecycle and Syncer benchmarks**.

Type: **Benchmark**.

Dependencies: **[Task 368][t368] and [Task 416][t416]**.

## Rationale

The existing Controller benchmarks time kvdb Wallets against live RPC and
therefore mix database work with process, socket, serialization, polling, and
mining latency. [Issue #1251][issue1251] needs a repeatable measurement of the
public start-to-synced workflow across kvdb, SQLite, and PostgreSQL before
backend-specific bottlenecks or tuning tasks can be justified.

## Details

- Add exactly `BenchmarkStartToSynced`, with explicit `kvdb`, `sqlite`, and
  `postgres` backend sub-benchmarks.
- Run the same contract and profiles on all three database backends. Do not
  skip kvdb inside the shared benchmark; keep backend setup outside the timer.
- Run `Cold` and `Warm` sub-benchmarks for the three stable sync profiles:
  `Sync-Empty-B1000-T100`, `Sync-Sparse-B1000-T100-M10`, and
  `Sync-Dense-B1000-T100-M100`.
- Use a deterministic in-process chain implementation containing all hashes,
  headers, filters, and blocks before the timer starts. It must perform no
  network I/O, mining, sleep, or random generation during a sample.
- Use `SyncMethodAuto` and the public `Manager.Start`/`Wallet.Info` boundary.
- In a cold sample, copy the base snapshot and construct the chain outside the
  timer; start the timer immediately before `NewManager`, include Manager
  Start, selecting the Wallet from its returned active set, and progress to the
  first `Info().Synced == true`. Stop the timer before result validation and
  Manager Stop.
- In a warm sample, construct a fresh Manager outside the timer; start
  immediately before `Manager.Start`, include Wallet activation and progress to
  the first `Info().Synced == true`, and stop the timer before validation and
  Manager Stop. Never reuse or restart a terminal Manager or Wallet pointer.
- Poll without an artificial sleep. The public `Info` calls used to observe
  completion are part of the measured workflow.
- Call `b.ReportAllocs`; assert exact final tip, wallet transaction count,
  UTXO count, and balance outside the timer.
- Exclude partial-restart, reorg, watched-outpoint, P1, runtime
  store diagnostics, profiling output, and performance changes. Create a
  diagnostic task only after results identify a concrete lower-level cost.

## Suggested fix

Add a focused Controller benchmark beside `controller_benchmark_test.go`.
Reuse the three-backend fixture lifecycle and add a small deterministic chain
dataset adapter dedicated to this benchmark. Keep existing comparison
benchmarks unchanged.

## Verification

- Run:

  ```text
  go test ./wallet -run '^$' \
    -bench '^BenchmarkStartToSynced$' -benchmem \
    -benchtime=1x -count=10 | tee sync.txt
  benchstat sync.txt
  ```

- For comparisons, capture identical commands from two revisions and run
  `benchstat sync-base.txt sync-change.txt`.
- Confirm each backend reports all six `Cold|Warm` and `Empty|Sparse|Dense`
  combinations with `B/op` and `allocs/op`.
- Instrument the deterministic chain in tests and assert zero network calls,
  zero sleeps, and no dataset construction after each timer starts.

## Definition of done

- The three exact datasets produce deterministic start-to-synced results on
  real kvdb, SQLite, and PostgreSQL Wallets.
- Cold and warm timer boundaries are explicit and stable.
- Setup and network latency are excluded, while the public lifecycle and
  completion observation are measured.
- Allocation results and repeatable benchstat capture are available.
- No runtime diagnostic, tuning, P1, or unrelated legacy comparison work is
  included.
- The PR contains only the start-to-synced benchmark outcome.

[t368]: 368-wallet-startup-shutdown.md
[t416]: 416-database-benchmark-fixtures.md
[issue1251]: https://github.com/btcsuite/btcwallet/issues/1251
