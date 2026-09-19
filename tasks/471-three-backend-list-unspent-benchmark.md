# Task 471 — measure ListUnspent on three real backends

Component: **wallet UtxoManager benchmarks and shared benchmark setup**.

Type: **Benchmark**.

Dependencies: **[Task 331][t331]**.

## Rationale

At `sql-wallet` revision `48e7f4b`, [BenchmarkListUnspentAPI][source] already
compares maintained and deprecated APIs, but its legacy fixture varies
accounts, addresses, and UTXOs together. It does not provide an equivalent
real-Manager baseline across kvdb, SQLite, and PostgreSQL.

On 2026-09-19, this task combines [416][t416] and [418][t418]. The first real
measurement exercises the shared setup directly, removing the need for a
separate setup-only example and smoke matrix.

## Details

- Modernize the existing ListUnspent benchmark instead of adding a competing
  baseline. Measure one fixed public request on real kvdb, SQLite, and
  PostgreSQL Wallets constructed through Manager.
- Reuse existing backend setup and deterministic data builders. Add only the
  private benchmark setup and cleanup this measurement requires; backend
  differences belong in setup and teardown, not the measured public call.
- Vary only live UTXO count across a few explicit sizes. Fix account selection,
  confirmation filters, and transaction shape; record supporting history
  without an independent history-size dimension.
- Use fixed seeds, times, hashes, scripts, and values. Equivalent inputs must
  produce the same logical Wallet state on all three backends.
- Seed and validate once per case, then repeatedly call ListUnspent on the
  same active Wallet. Do not reopen or reset between reads.
- Keep setup, full result validation, and cleanup outside timing. Validate
  count, order, outpoints, and values; fail on unexpected operation errors.
- Report time, allocations, exact UTXO count, and fixed request parameters.
- Leave the minimal setup reusable by [417][t417], [419][t419], and
  [420][t420]. Those tasks depend on the setup, not a performance result, and
  own their operation-specific data. Task 417 owns per-sample startup work.
- Exclude production changes, tuning, leases, mutation, concurrency, daemon
  matrices, deprecated-API comparisons, cold/warm modes, cache controls,
  snapshot/reset machinery, growth models, and generic benchmark runners.

## Suggested fix

Adapt the existing benchmark loop around real Manager/backend fixtures and a
short size table. Exercise shared setup with ListUnspent itself rather than
prebuilding later consumers or introducing a separate demonstration workload.

## Verification

- Run each size on all three backends and require equivalent public results
  outside timing. Reuse existing contract tests rather than repeat their
  account/address/transaction/UTXO/synchronization matrices.
- Confirm only repeated ListUnspent calls are timed and every result names
  its workload, time, and allocations.
- Exercise success and failed setup cleanup through this fixture, leaving no
  Manager, Wallet worker, connection, handle, or temporary database behind.

## Definition of done

One existing ListUnspent benchmark provides a reproducible three-backend
scaling baseline and minimal reusable setup, without production changes,
duplicate benchmarks, or a standalone setup-only test project.

[source]: https://github.com/btcsuite/btcwallet/blob/48e7f4b7e0f3808b5022fd817d12c6a4361bb9c1/wallet/utxo_manager_benchmark_test.go
[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t416]: ../archive/tasks/416-database-benchmark-fixtures.md
[t418]: ../archive/tasks/418-list-unspent-benchmark.md
[t417]: 417-start-to-synced-benchmark.md
[t419]: 419-address-read-benchmark.md
[t420]: 420-transaction-read-benchmark.md
