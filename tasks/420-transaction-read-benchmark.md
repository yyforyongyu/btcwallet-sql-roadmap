# Task 420 — measure transaction-read performance

Component: **wallet TxReader benchmarks**.

Type: **Benchmark**.

Dependencies: **[Task 416][t416]**.

## Rationale

GetTx and ListTxns measure point and range reads over transaction history.
Existing legacy fixtures and growth functions obscure the workload. A few
explicit history sizes are enough for an initial three-backend comparison.

## Details

- Measure each method separately on real kvdb, SQLite, and PostgreSQL Wallets
  using Task 416's setup.
- Vary only transaction count across a few explicit sizes. Fix account count,
  transaction shape, confirmed/unmined mix, the point-selection rule, and
  ListTxns filters. Include the whole seeded history in the list request.
- Seed and validate once per case, then repeatedly call the selected method
  on the same active Wallet. Do not reopen or reset between reads.
- Check the point result and complete list outside timing. Keep setup and
  cleanup outside timing and fail on unexpected operation errors.
- Report time, allocations, transaction count, mix, and request parameters.
- Exclude cold/warm modes, deprecated comparisons, labels, writes,
  invalidation, concurrency, tuning, and production changes.

## Suggested fix

Use a short explicit size table and a small loop for each method. Keep the
data and request visible without growth functions or profile abstractions.

## Verification

- Run each method and size on all three backends and require equivalent
  public results.
- Confirm only the selected operation is timed, each result names its workload,
  and every fixture is cleaned up.

## Definition of done

Both public transaction reads have simple, repeatable three-backend baselines
with transaction count as the scaling dimension and no production changes.

[t416]: 416-database-benchmark-fixtures.md
