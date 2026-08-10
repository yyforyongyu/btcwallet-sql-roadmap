# Open task catalog

The next free task number is **452**.

This is a link-only catalog. Each task file owns its scope and dependencies,
[QUEUE.md](../QUEUE.md) owns dependency phases, and
[ACTIVE.md](../ACTIVE.md) owns active tasks, claims, and in-progress PRs.
Grouping here does not create dependency or scheduling authority.

## Creation, Manager, and storage

- [Task 329](329-atomic-wallet-account-creation.md): commit SQL wallet genesis
  atomically.
- [Task 330](330-duplicate-wallet-create.md): reject duplicate wallet creation.
- [Task 331](331-manager-wallet-lifecycle.md): fence Manager admission and Store
  closure.
- [Task 332](332-watch-only-script-policy.md): define watch-only and
  script-import policy.
- [Task 333](333-postgres-manager-harness.md): add PostgreSQL Manager and harness
  support.
- [Task 340](340-durable-database-identity.md): bind SQL databases to schema and
  network identity.
- [Task 350](350-native-pgx-postgres.md): use native pgx for PostgreSQL storage.
- [Task 358](358-contextual-manager-operations.md): make Manager creation
  cancellable.
- [Task 359](359-revalidate-database-identity-design.md): revalidate database
  identity design.
- [Task 365](365-validate-wallet-genesis-keys.md): validate wallet genesis key
  identities.
- [Task 366](366-reconcile-sql-genesis-commit.md): reconcile ambiguous SQL
  genesis commits.
- [Task 368](368-wallet-startup-shutdown.md): serialize Wallet startup and
  shutdown.
- [Task 369](369-repeated-wallet-unlock.md): reject repeated Wallet unlocks
  without resetting timeout.
- [Task 379](379-manager-load-error-boundary.md): translate Manager load
  not-found errors.
- [Task 422](422-manager-load-cancellation.md): make Manager loading cancellable.

## Accounts, addresses, and recovery

- [Task 337](337-imported-xpub-derivation.md): restore SQL imported-XPub public
  derivation.
- [Task 341](341-public-role-contract.md): define semantic account identity.
- [Task 342](342-recovery-window-validation.md): accept bounded recovery windows.
- [Task 343](343-exact-account-creation.md): create next-or-exact root-derived
  accounts.
- [Task 344](344-bulk-address-allocation.md): allocate receiving-address batches
  atomically.
- [Task 345](345-durable-watch-registration.md): persist SQL account discovery
  mode.
- [Task 360](360-document-kvdb-adapter-identity.md): document kvdb adapter
  identity.
- [Task 361](361-kvdb-imported-xpub-derivation.md): support kvdb imported-XPub
  derivation.
- [Task 363](363-recovery-scan-work-bound.md): bound aggregate recovery
  derivation work.
- [Task 364](364-durable-watch-registration.md): persist ordered SQL live-watch
  obligations.
- [Task 377](377-account-manager-wallet-dto.md): make AccountManager results
  wallet-owned.
- [Task 378](378-account-manager-error-boundary.md): translate AccountManager
  backend errors.
- [Task 380](380-address-lookup-error-boundary.md): translate address lookup
  errors.
- [Task 381](381-recovery-adapter-identity.md): hide recovery adapter identity.
- [Task 387](387-account-creation-identity-adr.md): define custom and path-bound
  account identity.
- [Task 388](388-runtime-account-xpub-collisions.md): reject colliding runtime
  account XPub identities.
- [Task 389](389-runtime-account-commit-reconciliation.md): reconcile ambiguous
  runtime account commits.
- [Task 390](390-custom-scope-account-creation.md): create root-derived accounts
  in custom scopes.
- [Task 391](391-path-bound-watch-only-xpub-accounts.md): create exact path-bound
  watch-only XPub accounts.
- [Task 392](392-discovery-mode-account-creation.md): consume discovery mode in
  existing account creation.
- [Task 393](393-key-only-receiving-rejection.md): reject receiving allocation
  for key-only accounts.
- [Task 394](394-key-only-transaction-ownership.md): exclude key-only locators
  from wallet ownership.
- [Task 395](395-key-only-locator-allocation.md): allocate one durable key-only
  locator.
- [Task 396](396-oldest-unused-address.md): make NewAddress return the oldest
  unused address.
- [Task 397](397-semantic-recovery-targets-adr.md): define semantic-account
  recovery targets.
- [Task 398](398-semantic-account-recovery-targets.md): accept semantic account
  recovery targets.
- [Task 399](399-zero-lookahead-persisted-targets.md): isolate persisted targets
  in zero-lookahead recovery.
- [Task 400](400-live-watch-semantics-adr.md): separate live watches from
  historical recovery.
- [Task 401](401-live-tip-chain-watch.md): add an idempotent live-tip-only chain
  watch.
- [Task 402](402-replay-live-watch-obligations.md): replay live-watch obligations
  through the Syncer.
- [Task 403](403-raw-import-watch-obligations.md): persist raw imports with
  live-watch obligations.

## Chain and synchronization

- [Task 334](334-shared-chain-state.md): supersede ADR 0002 with SQL shared-
  chain ownership.
- [Task 370](370-chain-source-delivery.md): prepare ordered Manager chain
  delivery.
- [Task 371](371-canonical-chain-frontier.md): serialize canonical chain-frontier
  mutation.
- [Task 372](372-multi-wallet-rollback.md): prepare atomic multi-Wallet
  rollback.
- [Task 423](423-manager-chain-source-cutover.md): cut over SQL Manager chain
  notification ownership.

## Transactions, UTXOs, and PSBTs

- [Task 339](339-transaction-visibility.md): define public transaction visibility
  and status.
- [Task 346](346-psbt-funding-metadata.md): preserve PSBT metadata during
  funding.
- [Task 347](347-psbt-signing-authorization.md): authorize explicit PSBT
  signatures.
- [Task 348](348-watched-change-delivery.md): watch PSBT change before returning
  funding.
- [Task 349](349-transaction-invalidation.md): implement idempotent transaction
  invalidation.
- [Task 351](351-sql-utxo-integrity.md): bind SQL UTXOs to wallet tenancy.
- [Task 355](355-transaction-size-accounting.md): correct transaction size
  accounting.
- [Task 356](356-defer-change-script-allocation.md): defer change-script
  allocation until viable.
- [Task 357](357-transaction-amount-validation.md): validate transaction output
  and fee arithmetic.
- [Task 404](404-transaction-input-amounts.md): validate transaction input-source
  amounts.
- [Task 405](405-psbt-funding-lease-admission.md): reserve PSBT funding inputs
  atomically.
- [Task 406](406-psbt-funding-lease-release.md): release PSBT funding leases by
  exact token.
- [Task 407](407-psbt-finalization-authorization.md): authorize PSBT
  auto-finalization.
- [Task 408](408-direct-transaction-change-watch.md): watch direct transaction
  change before return.
- [Task 409](409-transaction-input-graph.md): normalize transaction input edges.
- [Task 410](410-transaction-graph-admission.md): coordinate transaction graph
  mutations.
- [Task 411](411-terminal-parent-rejection.md): reject children of terminal
  transactions.
- [Task 412](412-transaction-publication-admission.md): expose invalidation with
  publication ordering.
- [Task 413](413-utxo-spend-identity.md): constrain SQL UTXOs to exact spending
  inputs.
- [Task 414](414-utxo-amount-bounds.md): bound persisted SQL UTXO amounts.
- [Task 424](424-transaction-utxo-integrity-adr.md): define transaction and UTXO
  integrity ownership.
- [Task 425](425-separate-transaction-authoring-wrappers.md): isolate shared
  transaction authoring from wrapper watch and lease effects.
- [Task 426](426-sql-transaction-output-aggregates.md): validate persisted
  transaction output aggregates.

## Integration tests

- [Task 415](415-utxo-public-integration-tests.md): cover public UTXO contracts
  end to end.
- [Task 427](427-controller-lifecycle-credential-itests.md): cover remaining
  Controller credential contracts.
- [Task 428](428-controller-sync-info-itests.md): cover Controller resync and
  rescan contracts.
- [Task 429](429-account-mutation-itests.md): cover public account mutation
  contracts.
- [Task 430](430-account-query-itests.md): cover public account query contracts.
- [Task 431](431-address-allocation-lookup-itests.md): cover address allocation
  and lookup contracts.
- [Task 432](432-address-import-script-itests.md): cover address import and
  script contracts.
- [Task 433](433-wallet-sync-test-boundary.md): enforce the wallet sync test
  boundary.
- [Task 434](434-transaction-creation-itests.md): cover transaction creation
  contracts.
- [Task 435](435-transaction-publication-itests.md): cover transaction
  publication contracts.
- [Task 436](436-transaction-read-itests.md): cover transaction read contracts.
- [Task 437](437-transaction-label-itests.md): cover transaction label
  contracts.
- [Task 438](438-psbt-decoration-combination-itests.md): cover PSBT decoration
  and combination contracts.
- [Task 439](439-psbt-funding-itests.md): cover PSBT funding contracts.
- [Task 440](440-psbt-signing-finalization-itests.md): cover PSBT signing and
  finalization contracts.
- [Task 441](441-signer-derivation-ecdh-itests.md): cover public derivation and
  ECDH contracts.
- [Task 442](442-signer-signing-itests.md): cover public signing contracts.
- [Task 443](443-unsafe-signer-itests.md): cover UnsafeSigner extraction
  contracts.
- [Task 451](451-chain-version-itest-matrix.md): test supported chain daemon
  releases.

## Integration-test scenarios

- [Task 444](444-wallet-receive-reload-scenario.md): cover wallet receipt and
  durable reload.
- [Task 445](445-sql-account-signing-scenario.md): cover SQL account derivation
  and signing.
- [Task 446](446-transaction-spend-publish-scenario.md): cover transaction
  creation and publication.
- [Task 447](447-psbt-workflow-scenario.md): cover the complete PSBT workflow.
- [Task 448](448-sync-rollback-scenario.md): cover synchronization and rollback.
- [Task 449](449-multi-wallet-sql-isolation-scenario.md): cover multi-Wallet SQL
  isolation.
- [Task 450](450-bitcoind-compatibility-scenario.md): compare public wallet
  state with bitcoind.

## Secrets and credentials

- [Task 338](338-passphrase-result.md): latch accepted SQL passphrase results.
- [Task 352](352-kvdb-combined-passphrase-rotation.md): rotate kvdb passphrases
  atomically.
- [Task 354](354-vault-owned-secret-resolution.md): define Vault secret-operation
  ownership.
- [Task 362](362-kvdb-definitive-passphrase-result.md): return a definitive kvdb
  passphrase result.
- [Task 367](367-bound-sql-genesis-secrets.md): bound SQL genesis secret
  lifetime.
- [Task 382](382-vault-account-derivation.md): move account derivation into the
  Vault.
- [Task 383](383-vault-output-script-resolution.md): resolve output scripts
  through the Vault.
- [Task 384](384-vault-safe-signing.md): route safe signing through the Vault.
- [Task 385](385-unsafe-signer-key-extraction.md): isolate UnsafeSigner key
  extraction.
- [Task 386](386-remove-raw-wallet-secret-seams.md): remove raw Wallet secret
  seams.
- [Task 421](421-sql-passphrase-commit-reconciliation.md): reconcile SQL
  passphrase commits.

## Migration and retirement

- [Task 335](335-kvdb-to-sql-migration.md): validate an immutable legacy wallet
  source.
- [Task 373](373-extract-legacy-account-address-state.md): extract legacy account
  and address state.
- [Task 374](374-extract-legacy-transaction-sync-state.md): extract legacy
  transaction and sync state.
- [Task 375](375-import-legacy-snapshot-sqlite.md): import one legacy snapshot
  into SQLite.
- [Task 376](376-orchestrate-legacy-conversion.md): orchestrate and verify legacy
  conversion.
- [Task 353](353-retire-modern-kvdb-backend.md): retire kvdb from the modern
  wallet API.

## Benchmarks

- [Task 416](416-sqlite-benchmark-fixtures.md): build reusable SQLite benchmark
  fixtures.
- [Task 417](417-sqlite-start-to-synced-benchmark.md): measure SQLite
  start-to-synced performance.
- [Task 418](418-sqlite-list-unspent-benchmark.md): measure SQLite ListUnspent
  performance.
- [Task 419](419-sqlite-address-read-benchmark.md): measure SQLite address-read
  performance.
- [Task 420](420-sqlite-transaction-read-benchmark.md): measure SQLite
  transaction-read performance.

## Cleanup

- [Task 336](336-test-cleanup.md): remove redundant Wallet passphrase tests.
