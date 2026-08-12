# Task 416 — build reusable database benchmark fixtures

Component: **wallet benchmark infrastructure**.

Dependencies: **None**.

## Rationale

The current public API benchmarks build a legacy walletdb Wallet and seed
`waddrmgr` and `wtxmgr` directly. That fixture cannot provide one controlled
public-Manager workload across kvdb, SQLite, and PostgreSQL, or comparable
first-query versus warm-connection measurements. [Issue #1251][issue1251]
needs deterministic real-backend fixtures before individual P0 API
measurements can be compared within each backend.

## Details

- Add benchmark-only fixtures that create and load Wallets through real kvdb,
  SQLite, and PostgreSQL Managers and Stores.
- Register every shared fixture profile for all three database backends. Do
  not skip kvdb inside the shared fixture or benchmark paths; isolate only
  backend-specific setup and teardown.
- Own this common contract for every profile: one wallet named
  `benchmark-wallet` on regression net; deterministic 32-byte seed
  `00, 01, ... 1f`; fixed timestamps, block hashes, transaction hashes,
  scripts, and values; and one BIP84 scope with its default derived account
  unless a profile says otherwise.
- Use no imported addresses, leases, labels, concurrent writers, or random
  data. Every loaded Wallet uses the public Manager/Wallet boundary and its
  real persistent Store.
- Keep setup, embedded-database snapshot copying or equivalent PostgreSQL
  reset, deterministic chain construction, result checking, and teardown
  outside consumer timers.
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
- Checkpoint and close each populated embedded database before snapshotting
  it. Give each PostgreSQL iteration an equivalently isolated populated
  database so no consumer mutates its baseline.
- Provide explicit helpers for build, snapshot, cold reload, warm reuse,
  semantic validation, and cleanup.
- Add `TestBackendBenchmarkFixtureReload`, registered for kvdb, SQLite, and
  PostgreSQL, to prove that two reloads of each profile preserve wallet ID,
  address count, transaction count, live UTXO count, sync tip, and
  deterministic target identities.
- Add `BenchmarkBackendFixtureBuild` with backend and `Smoke` sub-benchmarks as
  calibration measurements. Every leaf must call `b.ReportAllocs`.
- For `BenchmarkBackendFixtureBuild`, start the timer immediately before
  creating the Manager and seeding `Smoke`; stop it after durable flush, close,
  and manifest calculation. Document that this calibration time is not part
  of any P0 API result.
- Document profile names, exact dimensions, cold/warm meanings, commands,
  timer rules, and benchstat capture in `wallet/README.md` or an adjacent
  benchmark document.
- Do not expose fixture types outside tests. Do not alter production behavior,
  queries, indexes, caching, connection defaults, or legacy fixtures.

## Suggested fix

Add a small backend benchmark fixture in package `wallet`. Give it explicit
kvdb, SQLite, and PostgreSQL factories that populate the same logical profile
through maintained Store-backed wallet operations or deterministic Store
fixture insertion. Return one immutable manifest plus a backend-specific
reload handle. Keep reload mechanics below the API benchmarks so every
consumer uses the same lifecycle and validation rules.

## Verification

- Run:

  ```text
  go test ./wallet -run '^TestBackendBenchmarkFixtureReload$'
  go test ./wallet -run '^$' \
    -bench '^BenchmarkBackendFixtureBuild$' -benchmem \
    -benchtime=1x -count=10 | tee fixture.txt
  benchstat fixture.txt
  ```

- For a before/after comparison, capture the same command from both revisions
  and run `benchstat fixture-base.txt fixture-change.txt`.
- Confirm every `BenchmarkBackendFixtureBuild/<backend>/Smoke` leaf reports
  `ns/op`, `B/op`, and `allocs/op` and no consumer includes fixture build time.
- Confirm fixture tests leave no kvdb or SQLite handles, SQLite sidecars, or
  PostgreSQL test databases and connections open.

## Definition of done

- Every named profile has one deterministic manifest and exact semantic
  validation.
- Cold reload and warm reuse are controlled by one shared helper.
- The calibration benchmark and reload test pass with real kvdb, SQLite, and
  PostgreSQL Stores and identical logical manifests.
- Allocation reporting and repeatable benchstat commands are documented.
- No wallet behavior, tuning, P1 benchmark, or runtime sync diagnostic is
  added.
- The PR contains only reusable three-backend benchmark infrastructure.

[issue1251]: https://github.com/btcsuite/btcwallet/issues/1251
[t417]: 417-sqlite-start-to-synced-benchmark.md
