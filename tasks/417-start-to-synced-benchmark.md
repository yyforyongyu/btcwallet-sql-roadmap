# Task 417 — measure start-to-synced performance

Component: **Manager lifecycle and Syncer benchmarks**.

Type: **Benchmark**.

Dependencies: **[Task 331][t331], [Task 471][t471], and [Task 455][t455]**.

## Rationale

Existing Controller benchmarks mix startup with live-chain RPC, mining, and
several setup variants. We need one understandable measurement of the public
Manager.Start-to-Wallet-synced workflow on kvdb, SQLite, and PostgreSQL.

## Details

- Use Task 471's real-backend setup and a deterministic in-process chain.
  Fix the Wallet/account counts and block-match pattern; vary only the block
  count across a short explicit list, including an empty history.
- Before each sample, prepare the same starting Wallet state and construct a
  fresh Manager. Never restart a terminal Manager or Wallet. Keep this local
  setup in the startup benchmark rather than adding reset modes to Task 471.
- Time Manager.Start through observation of the exact expected synchronized
  tip using public Wallet information. Include the completion observation in
  the measurement; exclude Manager construction, seeding, validation, and Stop.
- Prepare all chain data before timing. Do not mine, call a live chain daemon,
  generate random inputs, or add artificial sleeps inside a sample. Normal
  database I/O remains part of the measured operation on every backend.
- Verify final public state outside timing and report time, allocations, and
  the exact block count and match pattern.
- Add no cold/warm variants, cache controls, concurrency matrix, profiling,
  tuning, or production diagnostics.

## Suggested fix

Adapt the existing Controller benchmark source with one explicit startup
loop and untimed setup and teardown around each sample. Reuse existing public
synchronization observation without introducing a second benchmark lifecycle
or a generic timing framework.

## Verification

- Repeat each block-count case on kvdb, SQLite, and PostgreSQL and require the
  same exact final tip and Wallet-visible state.
- Confirm each sample starts from equivalent state, measures the same public
  workflow, and leaves no live Manager, Wallet, or backend resources.
- Report reproducible workload dimensions, time, and allocations.

## Definition of done

One start-to-synced measurement compares equivalent histories on all three
backends, with an obvious timing boundary and no production changes.

[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t471]: 471-three-backend-list-unspent-benchmark.md
[t455]: ../archive/tasks/455-exact-wallet-sync-status.md
