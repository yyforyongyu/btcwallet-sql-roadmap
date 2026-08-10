# Task 419 — measure SQLite address-read performance

Component: **wallet AddressManager benchmarks**.

Dependencies: **[Task 416][t416]**.

## Rationale

`ListAddresses` and `GetAddressInfo` are both P0 AddressManager reads over the
same immutable address dataset, but the current benchmarks seed a legacy
walletdb Wallet. The [Issue #1251][issue1251] SQL tranche can share fixture
construction while preserving separate API measurements for the list/balance
path and the single-address lookup path.

## Details

- Add exactly `BenchmarkSQLiteAddressReads` with separate `ListAddresses`
  and `GetAddressInfo` sub-benchmark trees.
- For both APIs, run `Cold` and `Warm` against `Addr-A100`, `Addr-A1000`, and
  `Addr-A10000`.
- Call `ListAddresses` with the default account name and
  `waddrmgr.WitnessPubKey`. Call `GetAddressInfo` with the deterministic
  middle address (`N/2`) from the same manifest.
- A cold iteration reloads outside the timer and times exactly one public API
  call. A warm sub-benchmark opens once, performs one untimed call to that same
  API, resets the timer, and times one call per iteration.
- Stop before verifying the full list count/order or the returned address,
  account, type, derivation path, and usedness.
- Call `b.ReportAllocs` in every leaf benchmark.
- Keep imported addresses, balances with transaction history, mutations,
  concurrent reads, tuning, and PostgreSQL out of this task.

## Suggested fix

Add SQL sub-benchmarks beside `address_manager_benchmark_test.go` and share one
profile loader between the two public API trees. Do not replace or rename the
existing legacy comparison benchmarks.

## Verification

- Run:

  ```text
  go test ./wallet -run '^$' \
    -bench '^BenchmarkSQLiteAddressReads$' -benchmem \
    -benchtime=3s -count=10 | tee address-reads.txt
  benchstat address-reads.txt
  ```

- Compare revisions with
  `benchstat address-reads-base.txt address-reads-change.txt`.
- Confirm all twelve API/cache/profile leaves report `ns/op`, `B/op`, and
  `allocs/op`.
- Confirm setup and validation do not run within either public-call timer.

## Definition of done

- Both P0 address reads use the same real SQLite snapshots and exact profile
  dimensions.
- Each API has an independent cold and warm timing boundary.
- Result validation proves list and lookup semantics outside the timer.
- Allocation reporting and repeatable benchstat capture are documented.
- No tuning, mutation path, P1 work, or PostgreSQL lane is included.
- The PR contains only the coherent AddressManager read benchmark outcome.

[t416]: 416-sqlite-benchmark-fixtures.md
[issue1251]: https://github.com/btcsuite/btcwallet/issues/1251
