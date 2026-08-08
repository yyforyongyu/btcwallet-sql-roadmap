# Task 416 — build reusable SQLite benchmark fixtures

Component: **wallet benchmark infrastructure**.

Dependencies: **None**.

## Rationale

The current public API benchmarks build a legacy walletdb Wallet and seed
`waddrmgr` and `wtxmgr` directly. That fixture cannot attribute results to the
SQLite Store and cannot provide controlled first-query versus reused-connection
measurements. [Issue #1251][issue1251] needs one deterministic real-SQL
fixture before individual P0 API measurements can be comparable.

## Details

- Add a benchmark-only fixture that creates and loads Wallets through a real
  SQLite Manager and Store.
- Own this common contract for every profile: one wallet named
  `benchmark-wallet` on regression net; deterministic 32-byte seed
  `00, 01, ... 1f`; fixed timestamps, block hashes, transaction hashes,
  scripts, and values; one SQLite connection; and one BIP84 scope with its
  default derived account unless a profile says otherwise.
- Use no imported addresses, leases, labels, concurrent writers, or random
  data. Every loaded Wallet uses the public Manager/Wallet boundary and a real
  SQLite Store.
- Keep setup, snapshot copying, deterministic chain construction, result
  checking, and teardown outside consumer timers.
- For the address, UTXO, and transaction read profiles, `Cold` means the first
  API call after opening a copied snapshot. `Warm` means the same loaded Wallet
  receives one untimed priming call before repeated timed calls.
- For sync profiles, define this lifecycle exception for
  [Task 417][t417]: `Cold` includes Manager construction, Load, Start, and
  synchronization, while `Warm` loads a stopped Wallet outside the timer and
  times its first Start and synchronization without a priming Start.
- Implement these exact address and calibration profiles:
  - `Smoke`: one account, 100 addresses, 100 confirmed transaction records,
    10 live UTXOs, and 10 blocks.
  - `Addr-A100`, `Addr-A1000`, and `Addr-A10000`: one account, respectively
    100, 1,000, or 10,000 external addresses, and no transaction history.
- Implement these exact UTXO profiles:
  - `Utxo-U100-H100`: 100 live UTXOs, 100 confirmed wallet transaction
    records, 100 address scripts, and 100 transactions per block.
  - `Utxo-U100-H10000`: 100 live UTXOs, 10,000 confirmed wallet transaction
    records, 100 address scripts, and 100 transactions per block.
  - `Utxo-U10000-H10000`: 10,000 live UTXOs, 10,000 confirmed wallet
    transaction records, 1,000 address scripts, and 100 transactions per
    block.
  - `Utxo-U10000-H100000`: 10,000 live UTXOs, 100,000 confirmed wallet
    transaction records, 1,000 address scripts, and 100 transactions per
    block.
- Give every UTXO-profile record one deterministic wallet-owned output. Build
  deterministic spend chains so exactly `U` outputs remain live and `H`
  confirmed wallet records remain relevant history. Include no unmined record
  in these profiles.
- Implement these exact transaction profiles:
  - `Tx-T100`: 90 confirmed and 10 unmined records.
  - `Tx-T1000`: 900 confirmed and 100 unmined records.
  - `Tx-T10000`: 9,000 confirmed and 1,000 unmined records.
- Give every transaction-profile record two inputs and two outputs, place 10
  confirmed records per block, and use 100 address scripts.
- Implement these exact sync profiles, each with 1,000 blocks, 100
  transactions per block, and 100 watched derived scripts:
  - `Sync-Empty-B1000-T100`: zero wallet matches.
  - `Sync-Sparse-B1000-T100-M10`: one match at each height 100, 200, through
    1,000.
  - `Sync-Dense-B1000-T100-M100`: one match at each height 10, 20, through
    1,000.
- Prebuild every sync header, compact filter, and block. Serve them through a
  deterministic in-process `chain.Interface` without sockets, sleeping, RPC
  serialization, or mining during a timed sample, and use `SyncMethodAuto`.
- Checkpoint and close each populated database before snapshotting it. Copy a
  closed snapshot for each cold iteration so no consumer mutates its baseline.
- Provide explicit helpers for build, snapshot, cold reload, warm reuse,
  semantic validation, and cleanup.
- Add `TestSQLiteBenchmarkFixtureReload` to prove that two reloads of each
  profile preserve wallet ID, address count, transaction count, live UTXO
  count, sync tip, and deterministic target identities.
- Add `BenchmarkSQLiteFixtureBuild` with the `Smoke` profile as a calibration
  benchmark. It must call `b.ReportAllocs`.
- For `BenchmarkSQLiteFixtureBuild`, start the timer immediately before
  creating the SQLite Manager and seeding `Smoke`; stop it after checkpoint,
  close, and manifest calculation. Document that this calibration time is not
  part of any P0 API result.
- Document profile names, exact dimensions, cold/warm meanings, commands,
  timer rules, and benchstat capture in `wallet/README.md` or an adjacent
  benchmark document.
- Do not expose fixture types outside tests. Do not alter production behavior,
  SQL queries, indexes, caching, connection defaults, or legacy fixtures.

## Suggested fix

Add a small `sqlite_benchmark_fixture_test.go` in package `wallet`. Build a
real SQLite Manager at a caller-provided path, use maintained Store-backed
wallet operations or deterministic Store fixture insertion to populate the
profile, close and checkpoint it, and return an immutable manifest plus
snapshot path. Keep reload mechanics below the API benchmarks so every
consumer uses the same lifecycle and validation rules.

## Verification

- Run:

  ```text
  go test ./wallet -run '^TestSQLiteBenchmarkFixtureReload$'
  go test ./wallet -run '^$' \
    -bench '^BenchmarkSQLiteFixtureBuild$' -benchmem \
    -benchtime=1x -count=10 | tee fixture.txt
  benchstat fixture.txt
  ```

- For a before/after comparison, capture the same command from both revisions
  and run `benchstat fixture-base.txt fixture-change.txt`.
- Confirm `BenchmarkSQLiteFixtureBuild/Smoke` reports `ns/op`, `B/op`, and
  `allocs/op` and that no consumer includes fixture build time.
- Confirm the fixture tests leave no SQLite handles or WAL/SHM sidecars open.

## Definition of done

- Every named profile has one deterministic manifest and exact semantic
  validation.
- Cold reload and warm reuse are controlled by one shared helper.
- The calibration benchmark and reload test pass with a real SQLite Store.
- Allocation reporting and repeatable benchstat commands are documented.
- No wallet behavior, tuning, P1 benchmark, PostgreSQL lane, or runtime sync
  diagnostic is added.
- The PR contains only reusable SQLite benchmark infrastructure.

[issue1251]: https://github.com/btcsuite/btcwallet/issues/1251
[t417]: 417-sqlite-start-to-synced-benchmark.md
