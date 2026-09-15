# Task 402 — establish live watches from persisted Wallet data

Component: **Wallet synchronization and live-watch readiness**.

Type: **Gap fix**.

Dependencies: **[Task 331][t331], [Task 334][t334], [Task 345][t345], and
[Task 401][t401]**.

## Rationale

Restart loses backend filters. Stored addresses identify required watches,
not live readiness; Task 401 still needs wiring into Wallet synchronization.

## Details

- Scope live-only behavior to btcd/bitcoind, including polling. Preserve
  Neutrino's registration/history unchanged; require no adapter fix.
- Reconstruct startup watches from existing address/script and outpoint rows;
  include previously watched addresses and exclude `NoChainSync` accounts.
- Wire `WatchAddrsFromTip` before Start/Create publishes operational Wallets.
  Preserve no-gap catch-up: neither downtime transactions nor incomplete
  filtered notifications may be skipped when advancing the scanned tip.
- Join startup snapshots with concurrent committed-address registration.
  Tasks 344/403 own producers; success waits for registration, not a new block,
  so future mempool payments are observed.
- Loss of the required chain connection/notification stream, or a backend
  registration failure leaving live readiness unproven, terminates the whole
  Manager through Task 331. Do not continue operating with a retry backlog.
  Validation, caller cancellation, and determinate request-local Store errors
  remain local; cancellation cannot abandon accepted registration work.
- Task 331 owns draining and joining before Store closure. A failing worker
  must not wait on shutdown that is waiting for that same worker.
- Preserve this behavior through Task 423 without requiring its later cutover.
- Keep production Wallet-owned. Align ADR 0015's failure/obligation policy and
  ADR 0005's live-only boundary, including the Neutrino exclusion, in that PR.
- Add no watch schema, completion flag, durable queue, public API, cache, or
  existing-mempool backfill. Consume Task 401 unchanged; start no new history.

## Suggested fix

Reuse Store reads, Syncer catch-up, and Task 331's terminal lifecycle.

## Verification

- On SQLite and PostgreSQL, crash after address commit and before registration;
  a fresh Manager reconstructs watches without pending/completed metadata.
- Race startup, address commits, block delivery, and registration. Prove no
  early operational result, missed interval, or falsely advanced scanned tip.
- With btcd/bitcoind, deliver a payment after registration but before mining;
  duplicates remain harmless and registration starts no historical rescan.
- Fail registration or close the required notification stream with two SQL
  Wallets. Both become terminal; accepted work joins before Store closes.
  Cover startup failure, concurrent Stop, and request-local cancellation too.
- Preserve kvdb format and shared three-database lifecycle/error assertions.

## Definition of done

For btcd/bitcoind, Wallet reconstructs watches and stops on lost readiness
without delivery persistence or a new API. Neutrino remains unchanged.

[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t334]: ../archive/tasks/334-shared-chain-state.md
[t345]: ../archive/tasks/345-durable-watch-registration.md
[t401]: ../archive/tasks/401-live-tip-chain-watch.md
