# Open task catalog

The next free task number is **493**.

This is a link-only catalog. Each task file owns its scope and dependencies,
[QUEUE.md](../QUEUE.md) owns delivery phases, and
[ACTIVE.md](../ACTIVE.md) owns active tasks, claims, and in-progress PRs.
Grouping here does not create dependency or scheduling authority.

Ad hoc PRs are recorded in [QUEUE.md](../QUEUE.md), not this numbered catalog.
They do not consume task numbers or require task files.

## Consumer adoption

- [Task 470](470-lnd-wallet-port.md): complete lnd Wallet runtime parity.

## Creation, Manager, and storage

- [Task 329](329-atomic-wallet-account-creation.md): commit SQL wallet genesis
  atomically.
- [Task 358](358-contextual-manager-operations.md): make Manager creation
  cancellable.
- [Task 457](457-process-local-manager-lease.md): enforce one process-local
  Manager per database.

## Accounts, addresses, and recovery

- [Task 397](397-semantic-recovery-targets-adr.md): define semantic-account
  recovery targets.
- [Task 398](398-semantic-account-recovery-targets.md): accept semantic account
  recovery targets.
- [Task 460](460-sequential-account-invalid-children.md): make sequential
  account allocation invalid-child safe.

## Chain and synchronization

- [Task 423](423-manager-chain-source-cutover.md): cut over Manager chain
  notification ownership.

## Transactions, UTXOs, and PSBTs

- [Task 411](411-terminal-parent-rejection.md): reject children of terminal
  transactions.
- [Task 426](426-sql-transaction-output-aggregates.md): validate persisted
  transaction output aggregates.

## Integration tests

Shared public contracts run on kvdb, SQLite, and PostgreSQL. A task may exclude
a backend only when that backend cannot represent the scenario, and must state
that structural exclusion explicitly.

- [Task 440](440-psbt-signing-finalization-itests.md): cover PSBT signing and
  finalization contracts.

## Integration-test scenarios

- [Task 449](449-multi-wallet-sql-isolation-scenario.md): cover multi-Wallet SQL
  isolation.
- [Task 472](472-multi-wallet-rescan-isolation-scenario.md): isolate historical
  rescans from live Wallets.

## Secrets and credentials

- [Task 382](382-vault-account-derivation.md): move account derivation into the
  Vault.
- [Task 384](384-vault-safe-signing.md): route safe signing through the Vault.
- [Task 385](385-unsafe-signer-key-extraction.md): isolate UnsafeSigner key
  extraction.

## Migration

- [Task 335](335-kvdb-to-sql-migration.md): validate an immutable bbolt wallet
  source.
- [Task 373](373-extract-legacy-account-address-state.md): extract legacy
  wallet, account, and secret state.
- [Task 374](374-extract-legacy-transaction-sync-state.md): extract legacy
  transaction and sync state.
- [Task 375](375-import-legacy-snapshot-sqlite.md): import one legacy snapshot
  into SQLite.
- [Task 376](376-orchestrate-legacy-conversion.md): convert a legacy wallet
  through Manager.Create.
- [Task 473](473-extract-legacy-address-script-state.md): extract legacy address
  and script state.
- [Task 476](476-read-legacy-sql-walletdb-source.md): read legacy SQLite and
  PostgreSQL walletdb sources.
- [Task 477](477-import-legacy-snapshot-postgres.md): import a legacy snapshot
  into native PostgreSQL.
- [Task 485](485-convert-legacy-sql-wallets-through-manager.md): convert legacy
  SQL wallets through Manager.

## Benchmarks

Reuse the SQL fixtures delivered by [PR #1448][pr1448]. Each benchmark
owns its data and baseline alignment. Preserve existing kvdb comparisons
under the [scoped execution exception](../AGENTS.md#scope-rules).
Correctness validation belongs to unit/integration tests. Benchmarks retain
setup/operation error checks and matching workloads and timing boundaries.
Start with the smallest dataset and one selected case: build the code, run
relevant tests, and smoke-run that case. Present the code and results for user
approval before running full datasets, all cases, or full profile captures;
proceed only after explicit approval.
Keep one short file per method under btcwallet's `docs/developer/benchmark/`:
start with the method title and a succinct description of how it is run, then
`### Benchmark Results`, `### Data`, and `### Commands`, as in
[Task 471's example](471-three-backend-list-unspent-benchmark.md#benchmark-record).
Include exact benchmark and profile commands with Docker invocation and working
directories, method/commit SHAs, methodology/Docker environment, a data table,
and one ns/op, B/op, allocs/op results table with percentage changes against
legacy.
New SQL benchmarks live in `wallet/bench`; all four cases run in a common
Docker environment with separate CPU/memory pprof captures. No CPU time/RSS
reporting or new reporting framework is required.

- [Task 417](417-start-to-synced-benchmark.md): measure start-to-synced
  performance.
- [Task 419](419-address-read-benchmark.md): measure address-read
  performance.
- [Task 420](420-transaction-read-benchmark.md): measure transaction-read
  performance.
- [Task 471](471-three-backend-list-unspent-benchmark.md): measure ListUnspent
  on SQL with existing kvdb baselines.
- [Task 490](490-get-utxo-benchmark.md): measure GetUtxo performance.
- [Task 491](491-get-account-benchmark.md): measure GetAccount performance,
  including balance retrieval.
- [Task 492](492-list-accounts-benchmark.md): measure ListAccounts performance,
  including balances.

[pr1448]: https://github.com/btcsuite/btcwallet/pull/1448
