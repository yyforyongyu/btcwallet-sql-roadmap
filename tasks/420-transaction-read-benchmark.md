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

- Follow the [scoped benchmark exception][backend-rule] and [Task 471][t471]'s
  matching and reporting contract, reusing its minimal SQL setup.
- Measure each method separately with one representative existing workload.
  Match actual persisted history count, account/address state, input/output
  shape, confirmed/unmined mix, point selection, and ListTxns filters and
  expected content. Include the whole seeded history in the list request.
  A few explicit sizes may vary history count alone, keeping complexity fixed.
- Seed and validate once per case, then repeatedly call the selected method
  on the same active Wallet. Do not reopen or reset between reads.
- Validate actual state, the point result, and complete list outside timing.
  Keep setup and cleanup outside timing and fail on unexpected operation errors.
- Own the selected history cases' necessary baseline alignment and export;
  use 471's contract for identity, provenance, comparisons, and missing results.
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
- Export operation-specific results under Task 471's reporting contract.

## Definition of done

Both public transaction reads have focused SQL measurements matched to
selected existing workloads, with results under 471's reporting contract,
preserved kvdb comparisons, and no production changes.

[backend-rule]: ../AGENTS.md#scope-rules
[t471]: 471-three-backend-list-unspent-benchmark.md
