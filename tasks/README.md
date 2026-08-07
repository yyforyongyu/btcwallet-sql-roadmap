# Active task catalog

The next free task number is **365**.

This is a link-only catalog. Each task file owns its scope and dependencies,
[QUEUE.md](../QUEUE.md) owns recommended order, and
[ACTIVE.md](../ACTIVE.md) owns assignments and active PRs. Grouping here does
not create dependency or scheduling authority.

## Creation, Manager, and storage

- [Task 329](329-atomic-wallet-account-creation.md): make wallet and account
  creation atomic.
- [Task 330](330-duplicate-wallet-create.md): reject duplicate wallet
  creation.
- [Task 331](331-manager-wallet-lifecycle.md): harden Manager and Wallet
  lifecycle behavior.
- [Task 332](332-watch-only-script-policy.md): define watch-only and
  script-import policy.
- [Task 333](333-postgres-manager-harness.md): add PostgreSQL Manager and
  harness support.
- [Task 340](340-durable-database-identity.md): bind SQL databases to schema
  and network identity.
- [Task 350](350-native-pgx-postgres.md): use native pgx for PostgreSQL
  storage.
- [Task 358](358-contextual-manager-operations.md): make Manager creation and
  loading cancellable.
- [Task 359](359-revalidate-database-identity-design.md): revalidate database
  identity design.

## Accounts, addresses, and recovery

- [Task 337](337-imported-xpub-derivation.md): restore SQL imported-XPub
  public derivation.
- [Task 341](341-public-role-contract.md): make public role contracts
  wallet-owned.
- [Task 342](342-recovery-window-validation.md): accept bounded recovery
  windows.
- [Task 343](343-exact-account-creation.md): support exact and custom account
  creation.
- [Task 344](344-bulk-address-allocation.md): add atomic address and key
  allocation.
- [Task 345](345-durable-watch-registration.md): persist SQL account discovery
  mode.
- [Task 360](360-document-kvdb-adapter-identity.md): document kvdb adapter
  identity.
- [Task 361](361-kvdb-imported-xpub-derivation.md): support kvdb imported-XPub
  derivation.
- [Task 363](363-recovery-scan-work-bound.md): bound recovery scan work.
- [Task 364](364-durable-watch-registration.md): deliver durable live-watch
  registrations.

## Chain and synchronization

- [Task 334](334-shared-chain-state.md): coordinate shared chain state and
  rollback.

## Transactions, UTXOs, and PSBTs

- [Task 339](339-transaction-visibility.md): define public transaction
  visibility and status.
- [Task 346](346-psbt-funding-metadata.md): preserve PSBT metadata during
  funding.
- [Task 347](347-psbt-signing-authorization.md): complete PSBT signing
  authorization.
- [Task 348](348-watched-change-delivery.md): register change before returning
  funded results.
- [Task 349](349-transaction-invalidation.md): expose idempotent transaction
  invalidation.
- [Task 351](351-sql-utxo-integrity.md): enforce SQL UTXO integrity by
  constraint.
- [Task 355](355-transaction-size-accounting.md): correct transaction size
  accounting.
- [Task 356](356-defer-change-script-allocation.md): defer change-script
  allocation until viable.
- [Task 357](357-transaction-amount-validation.md): validate transaction
  monetary aggregates.

## Secrets and credentials

- [Task 338](338-passphrase-result.md): return a definitive SQL
  passphrase-change result.
- [Task 352](352-kvdb-combined-passphrase-rotation.md): make kvdb combined
  rotation atomic.
- [Task 354](354-vault-owned-secret-resolution.md): move secret resolution
  behind the Vault.
- [Task 362](362-kvdb-definitive-passphrase-result.md): return a definitive
  kvdb passphrase result.

## Migration and retirement

- [Task 335](335-kvdb-to-sql-migration.md): convert existing kvdb wallets to
  SQL.
- [Task 353](353-retire-modern-kvdb-backend.md): retire kvdb from the modern
  wallet API.

## Cleanup

- [Task 336](336-test-cleanup.md): remove redundant Wallet tests.
