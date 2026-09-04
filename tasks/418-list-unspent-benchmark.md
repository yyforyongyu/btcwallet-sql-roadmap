# Task 418 — measure ListUnspent performance

Component: **wallet UtxoManager benchmarks**.

Type: **Benchmark**.

Dependencies: **[Task 416][t416]**.

## Rationale

ListUnspent loads, enriches, filters, and sorts UTXOs. Existing benchmarks vary
accounts, addresses, and history together, making the result hard to explain.
One fixed request over a few explicit live-set sizes provides a useful baseline.

## Details

- Measure the same public ListUnspent request on real kvdb, SQLite, and
  PostgreSQL Wallets using Task 416's setup.
- Vary only live UTXO count across a few explicit sizes. Fix account selection,
  confirmation filters, and transaction shape; record the supporting history
  without adding an independent history-size matrix.
- Seed and validate once per case, then repeatedly call ListUnspent on the
  same active Wallet. Do not reopen or reset it between reads.
- Keep setup, full result checks, and cleanup outside timing. Validate count,
  order, outpoint identity, and values; fail on unexpected operation errors.
- Report time, allocations, exact UTXO count, and request parameters.
- Exclude cold/warm modes, deprecated APIs, leases, mutation, concurrency,
  query changes, indexes, tuning, and production fixes.

## Suggested fix

Use a small explicit size table and an ordinary benchmark loop containing one
public call. Keep the workload readable without growth functions or profile
abstractions.

## Verification

- Run each size on all three backends and require the same public results.
- Confirm only the repeated operation is timed and every result identifies
  its dataset size and fixed request.
- Verify complete cleanup and report time and allocations.

## Definition of done

ListUnspent has a simple, repeatable three-backend scaling baseline without
changing production behavior.

[t416]: 416-database-benchmark-fixtures.md
