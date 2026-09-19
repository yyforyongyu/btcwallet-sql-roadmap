# Task 419 — measure address-read performance

Component: **wallet AddressManager benchmarks**.

Type: **Benchmark**.

Dependencies: **[Task 471][t471]**.

## Rationale

ListAddresses and GetAddressInfo measure list and point reads over the same
address state. Existing legacy comparisons and coupled growth variables do
not provide an understandable three-backend baseline for these public APIs.

## Details

- Measure each method in its own benchmark on real kvdb, SQLite, and
  PostgreSQL Wallets using Task 471's setup.
- Vary only address count across a few explicit sizes. Fix account count,
  address type, request filters, and the rule selecting the point lookup.
- Seed and validate once per case, then repeatedly call the selected method
  on the same active Wallet. Do not reopen or reset between reads.
- Validate complete list semantics and the point result outside timing. Keep
  setup and cleanup outside timing and fail on unexpected operation errors.
- Report time, allocations, address count, and the fixed request parameters.
- Exclude cold/warm modes, deprecated comparisons, imports, mutations,
  transaction-backed balances, concurrency, tuning, and production changes.

## Suggested fix

Modernize the existing `BenchmarkListAddressesAPI` and
`BenchmarkGetAddressInfoAPI` loops rather than adding a second benchmark
family. Use one small size table and the same deterministic data for separate
list and point measurements, without a workload framework.

## Verification

- Run each method and size on all three backends and require equivalent
  public results.
- Confirm only the selected operation is timed, each result names its workload,
  and every fixture is cleaned up.

## Definition of done

Both public address reads have simple, repeatable three-backend measurements
with address count as the only scaling dimension and no production changes.

[t471]: 471-three-backend-list-unspent-benchmark.md
