# Open task catalog

The next free task number is **475**.

This is a link-only catalog. Each task file owns its scope and dependencies,
[QUEUE.md](../QUEUE.md) owns delivery phases, and
[ACTIVE.md](../ACTIVE.md) owns active tasks, claims, and in-progress PRs.
Grouping here does not create dependency or scheduling authority.

Ad hoc PRs are recorded in [QUEUE.md](../QUEUE.md), not this numbered catalog.
They do not consume task numbers or require task files.

## Consumer adoption

- [Task 470](470-lnd-wallet-port.md): port lnd to maintained Wallet APIs.

## Creation, Manager, and storage

- [Task 329](329-atomic-wallet-account-creation.md): commit SQL wallet genesis
  atomically.
- [Task 330](330-duplicate-wallet-create.md): reject duplicate wallet creation.
- [Task 350](350-native-pgx-postgres.md): use native pgx for PostgreSQL storage.
- [Task 358](358-contextual-manager-operations.md): make Manager creation
  cancellable.
- [Task 365](365-validate-wallet-genesis-keys.md): validate wallet genesis key
  identities.
- [Task 454](454-postgres-transaction-isolation.md): make PostgreSQL
  transaction isolation explicit.
- [Task 457](457-process-local-manager-lease.md): enforce one process-local
  Manager per database.

## Accounts, addresses, and recovery

- [Task 344](344-bulk-address-allocation.md): allocate receiving-address batches
  atomically.
- [Task 388](388-runtime-account-xpub-collisions.md): reject colliding runtime
  account XPub identities.
- [Task 389](389-runtime-account-commit-reconciliation.md): report uncertain
  account commits truthfully.
- [Task 391](391-path-bound-watch-only-xpub-accounts.md): create exact path-bound
  XPub accounts.
- [Task 395](395-key-only-locator-allocation.md): add `AllocateNextKey` for
  durable key allocation.
- [Task 396](396-oldest-unused-address.md): make NewAddress return the oldest
  unused address.
- [Task 397](397-semantic-recovery-targets-adr.md): define semantic-account
  recovery targets.
- [Task 398](398-semantic-account-recovery-targets.md): accept semantic account
  recovery targets.
- [Task 402](402-replay-live-watch-obligations.md): establish live watches from
  persisted Wallet data.
- [Task 403](403-raw-import-watch-obligations.md): register raw imports before
  public success.
- [Task 460](460-sequential-account-invalid-children.md): make sequential
  account allocation invalid-child safe.

## Chain and synchronization

- [Task 370](370-chain-source-delivery.md): fan out ordered Manager chain
  events.
- [Task 372](372-multi-wallet-rollback.md): coordinate multi-Wallet rollback.
- [Task 423](423-manager-chain-source-cutover.md): cut over Manager chain
  notification ownership.

## Transactions, UTXOs, and PSBTs

- [Task 347](347-psbt-signing-authorization.md): authorize existing PSBT
  signatures.
- [Task 356](356-defer-change-script-allocation.md): defer change-script
  allocation until viable.
- [Task 405](405-psbt-funding-lease-admission.md): reserve PSBT funding inputs
  atomically.
- [Task 406](406-psbt-funding-lease-release.md): clean up failed PSBT funding
  leases.
- [Task 407](407-psbt-finalization-authorization.md): authorize PSBT
  auto-finalization.
- [Task 410](410-transaction-graph-admission.md): coordinate transaction graph
  mutations.
- [Task 411](411-terminal-parent-rejection.md): reject children of terminal
  transactions.
- [Task 424](424-transaction-utxo-integrity-adr.md): define transaction and UTXO
  integrity ownership.
- [Task 426](426-sql-transaction-output-aggregates.md): validate persisted
  transaction output aggregates.
- [Task 469](469-sign-psbt-with-effective-keys.md): sign PSBT inputs with
  effective keys.

- [Task 474](474-authored-change-watch.md): watch authored change before return.

## Integration tests

Shared public contracts run on kvdb, SQLite, and PostgreSQL. A task may exclude
a backend only when that backend cannot represent the scenario, and must state
that structural exclusion explicitly.

- [Task 428](428-controller-sync-info-itests.md): cover Controller resync and
  rescan contracts.
- [Task 430](430-account-query-itests.md): cover public account query contracts.
- [Task 431](431-address-allocation-lookup-itests.md): cover address allocation
  and lookup contracts.
- [Task 432](432-address-import-script-itests.md): cover address import and
  script contracts.
- [Task 438](438-psbt-decoration-combination-itests.md): cover PSBT decoration
  and combination contracts.
- [Task 439](439-psbt-funding-itests.md): cover PSBT funding contracts.
- [Task 440](440-psbt-signing-finalization-itests.md): cover PSBT signing and
  finalization contracts.
- [Task 442](442-signer-signing-itests.md): cover public signing contracts.
- [Task 451](451-chain-version-itest-matrix.md): test supported chain daemon
  releases.
- [Task 461](461-address-batch-itests.md): cover public address batch
  contracts.
- [Task 462](462-account-mutation-error-itests.md): complete missing account
  mutation-error evidence.
- [Task 463](463-path-bound-utxo-identity-itests.md): verify path-bound UTXO
  account identity.
- [Task 464](464-no-chain-sync-account-query-itests.md): verify true-value
  list and path-bound account queries.
- [Task 465](465-new-address-result-itests.md): verify durable key allocation
  results.
- [Task 466](466-oldest-unused-address-itests.md): verify final oldest-unused
  allocation.

## Integration-test scenarios

- [Task 444](444-wallet-receive-reload-scenario.md): cover new-account receipt
  and durable reload.
- [Task 445](445-account-signing-scenario.md): cover new-account signing
  identity.
- [Task 446](446-transaction-spend-publish-scenario.md): cover authored-spend
  publication and reload.
- [Task 447](447-psbt-workflow-scenario.md): cover the complete PSBT workflow.
- [Task 448](448-sync-rollback-scenario.md): cover synchronization and rollback.
- [Task 449](449-multi-wallet-sql-isolation-scenario.md): cover multi-Wallet SQL
  isolation.
- [Task 450](450-bitcoind-compatibility-scenario.md): compare public wallet
  state with bitcoind.
- [Task 472](472-multi-wallet-rescan-isolation-scenario.md): isolate historical
  rescans from live Wallets.

## Secrets and credentials

- [Task 338](338-passphrase-result.md): latch accepted passphrase results.
- [Task 354](354-vault-owned-secret-resolution.md): define Vault secret-operation
  ownership.
- [Task 367](367-bound-sql-genesis-secrets.md): bound SQL genesis secret
  lifetime.
- [Task 369](369-repeated-wallet-unlock.md): reject repeated Wallet unlocks
  without resetting timeout.
- [Task 382](382-vault-account-derivation.md): move account derivation into the
  Vault.
- [Task 383](383-vault-output-script-resolution.md): resolve output scripts
  through the Vault.
- [Task 384](384-vault-safe-signing.md): route safe signing through the Vault.
- [Task 385](385-unsafe-signer-key-extraction.md): isolate UnsafeSigner key
  extraction.

## Migration

- [Task 335](335-kvdb-to-sql-migration.md): validate an immutable legacy wallet
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

## Benchmarks

Task 471 exercises minimal shared kvdb, SQLite, and PostgreSQL setup with the
first real ListUnspent measurement. Other consumers reuse that setup and own
their operation-specific data. Each uses a few explicit sizes and a simple
loop, without a cold/warm matrix or workload framework.

- [Task 417](417-start-to-synced-benchmark.md): measure start-to-synced
  performance.
- [Task 419](419-address-read-benchmark.md): measure address-read
  performance.
- [Task 420](420-transaction-read-benchmark.md): measure transaction-read
  performance.
- [Task 471](471-three-backend-list-unspent-benchmark.md): measure ListUnspent
  on three real backends with reusable setup.
