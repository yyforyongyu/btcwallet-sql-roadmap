# Active execution state

Every active task appears exactly once as a primary bullet below. The layers
are derived from direct active-task prerequisites. They describe state; they
do not rank work, and a task does not wait for an entire earlier layer. It
waits only for the dependencies and gates named in its task file.

Individual task files remain authoritative for scope and dependency edges.
[ACTIVE.md](ACTIVE.md) remains authoritative for assignments, claims, branches,
and PRs. Assignment labels here are a derived snapshot without owner or branch
details.

## Current state

These tasks have no incomplete active-task prerequisite. External coordination
still applies where recorded in [ACTIVE.md](ACTIVE.md).

### Active

- [Task 332](tasks/332-watch-only-script-policy.md) — define watch-only and
  script-import policy.
- [Task 333](tasks/333-postgres-manager-harness.md) — add PostgreSQL Manager
  and harness support.

### Assigned but unclaimed

- [Task 331](tasks/331-manager-wallet-lifecycle.md) — harden Manager and Wallet
  lifecycle behavior.
- [Task 350](tasks/350-native-pgx-postgres.md) — use native pgx for PostgreSQL
  storage. This is the only active task with neither an active-task
  prerequisite nor an active-task consumer.

### Unassigned and dependency-complete

- [Task 339](tasks/339-transaction-visibility.md) — define public transaction
  visibility and status.
- [Task 352](tasks/352-kvdb-combined-passphrase-rotation.md) — make kvdb
  combined rotation atomic.
- [Task 355](tasks/355-transaction-size-accounting.md) — correct transaction
  size accounting.
- [Task 357](tasks/357-transaction-amount-validation.md) — validate
  transaction monetary aggregates.
- [Task 359](tasks/359-revalidate-database-identity-design.md) — revalidate
  database identity design.
- [Task 360](tasks/360-document-kvdb-adapter-identity.md) — document kvdb
  adapter identity.

## Layer 1

- [Task 330](tasks/330-duplicate-wallet-create.md) — waits for
  [Task 331](tasks/331-manager-wallet-lifecycle.md) and
  [Task 333](tasks/333-postgres-manager-harness.md).
- [Task 338](tasks/338-passphrase-result.md) — waits for
  [Task 331](tasks/331-manager-wallet-lifecycle.md).
- [Task 340](tasks/340-durable-database-identity.md) — assigned; waits for
  [Task 359](tasks/359-revalidate-database-identity-design.md).
- [Task 341](tasks/341-public-role-contract.md) — waits for
  [Task 360](tasks/360-document-kvdb-adapter-identity.md).
- [Task 346](tasks/346-psbt-funding-metadata.md) — waits for
  [Task 357](tasks/357-transaction-amount-validation.md).
- [Task 354](tasks/354-vault-owned-secret-resolution.md) — waits for an
  accepted ADR 0010 amendment and
  [Task 352](tasks/352-kvdb-combined-passphrase-rotation.md) while modern kvdb
  remains. If kvdb retires first, it waits instead for
  [Task 353](tasks/353-retire-modern-kvdb-backend.md) and moves later.
- [Task 356](tasks/356-defer-change-script-allocation.md) — waits for
  [Task 355](tasks/355-transaction-size-accounting.md) and
  [Task 357](tasks/357-transaction-amount-validation.md), including their
  released nested-module changes.

## Layer 2

- [Task 334](tasks/334-shared-chain-state.md) — assigned; waits for
  [Task 333](tasks/333-postgres-manager-harness.md),
  [Task 340](tasks/340-durable-database-identity.md), and an accepted ADR
  superseding ADR 0002.
- [Task 337](tasks/337-imported-xpub-derivation.md) — waits for
  [Task 341](tasks/341-public-role-contract.md).
- [Task 342](tasks/342-recovery-window-validation.md) — waits for
  [Task 341](tasks/341-public-role-contract.md).
- [Task 345](tasks/345-durable-watch-registration.md) — waits for
  [Task 340](tasks/340-durable-database-identity.md).
- [Task 362](tasks/362-kvdb-definitive-passphrase-result.md) — waits for
  [Task 338](tasks/338-passphrase-result.md) and
  [Task 352](tasks/352-kvdb-combined-passphrase-rotation.md).
- [Task 365](tasks/365-validate-wallet-genesis-keys.md) — waits for
  [Task 332](tasks/332-watch-only-script-policy.md) and
  [Task 341](tasks/341-public-role-contract.md).

## Layer 3

- [Task 329](tasks/329-atomic-wallet-account-creation.md) — assigned; waits for
  [Task 332](tasks/332-watch-only-script-policy.md),
  [Task 337](tasks/337-imported-xpub-derivation.md), and
  [Task 345](tasks/345-durable-watch-registration.md).
- [Task 335](tasks/335-kvdb-to-sql-migration.md) — waits for
  [Task 332](tasks/332-watch-only-script-policy.md),
  [Task 337](tasks/337-imported-xpub-derivation.md),
  [Task 340](tasks/340-durable-database-identity.md), and
  [Task 345](tasks/345-durable-watch-registration.md).
- [Task 336](tasks/336-test-cleanup.md) — waits for
  [Task 338](tasks/338-passphrase-result.md),
  [Task 354](tasks/354-vault-owned-secret-resolution.md), and
  [Task 362](tasks/362-kvdb-definitive-passphrase-result.md) while modern kvdb
  remains. If kvdb retires first, it waits instead for
  [Task 353](tasks/353-retire-modern-kvdb-backend.md) and may move later.
- [Task 349](tasks/349-transaction-invalidation.md) — waits for
  [Task 334](tasks/334-shared-chain-state.md),
  [Task 339](tasks/339-transaction-visibility.md),
  [Task 340](tasks/340-durable-database-identity.md), and an accepted ADR 0006
  amendment making normalized inputs authoritative.
- [Task 361](tasks/361-kvdb-imported-xpub-derivation.md) — waits for
  [Task 337](tasks/337-imported-xpub-derivation.md).
- [Task 364](tasks/364-durable-watch-registration.md) — waits for
  [Task 345](tasks/345-durable-watch-registration.md) and accepted ADR 0004 and
  ADR 0005 amendments for live-watch delivery.

## Layer 4

- [Task 343](tasks/343-exact-account-creation.md) — waits for
  [Task 332](tasks/332-watch-only-script-policy.md),
  [Task 337](tasks/337-imported-xpub-derivation.md),
  [Task 340](tasks/340-durable-database-identity.md),
  [Task 341](tasks/341-public-role-contract.md),
  [Task 345](tasks/345-durable-watch-registration.md),
  [Task 354](tasks/354-vault-owned-secret-resolution.md),
  [Task 364](tasks/364-durable-watch-registration.md), and accepted ADR 0012
  and ADR 0013 amendments.
- [Task 344](tasks/344-bulk-address-allocation.md) — waits for
  [Task 337](tasks/337-imported-xpub-derivation.md) and
  [Task 364](tasks/364-durable-watch-registration.md).
- [Task 348](tasks/348-watched-change-delivery.md) — waits for
  [Task 346](tasks/346-psbt-funding-metadata.md),
  [Task 355](tasks/355-transaction-size-accounting.md),
  [Task 356](tasks/356-defer-change-script-allocation.md), and
  [Task 364](tasks/364-durable-watch-registration.md).
- [Task 351](tasks/351-sql-utxo-integrity.md) — waits for
  [Task 340](tasks/340-durable-database-identity.md),
  [Task 349](tasks/349-transaction-invalidation.md), and an accepted ADR 0006
  amendment or supersession covering declarative UTXO ownership.
- [Task 353](tasks/353-retire-modern-kvdb-backend.md) — waits for
  [Task 335](tasks/335-kvdb-to-sql-migration.md), production conversion
  evidence, and an explicit consumer cutover decision.
- [Task 363](tasks/363-recovery-scan-work-bound.md) — waits for
  [Task 337](tasks/337-imported-xpub-derivation.md),
  [Task 342](tasks/342-recovery-window-validation.md),
  [Task 345](tasks/345-durable-watch-registration.md),
  [Task 364](tasks/364-durable-watch-registration.md), an accepted ADR 0004
  amendment, and [Task 361](tasks/361-kvdb-imported-xpub-derivation.md) while
  modern kvdb remains. If kvdb retires first, it waits instead for
  [Task 353](tasks/353-retire-modern-kvdb-backend.md) and moves later.
- [Task 366](tasks/366-reconcile-sql-genesis-commit.md) — waits for
  [Task 329](tasks/329-atomic-wallet-account-creation.md).
- [Task 367](tasks/367-bound-sql-genesis-secrets.md) — waits for
  [Task 329](tasks/329-atomic-wallet-account-creation.md).

## Layer 5

- [Task 347](tasks/347-psbt-signing-authorization.md) — waits for
  [Task 343](tasks/343-exact-account-creation.md),
  [Task 346](tasks/346-psbt-funding-metadata.md), and
  [Task 354](tasks/354-vault-owned-secret-resolution.md).
- [Task 358](tasks/358-contextual-manager-operations.md) — waits for
  [Task 331](tasks/331-manager-wallet-lifecycle.md),
  [Task 366](tasks/366-reconcile-sql-genesis-commit.md), and
  [Task 367](tasks/367-bound-sql-genesis-secrets.md).

## Maintaining this map

When a task completes or its dependency contract changes, refresh its current
state and every downstream layer affected by that edge. Preserve one primary
entry per active task. Layers never authorize claiming a task before all hard
dependencies are complete; consult the task file and [ACTIVE.md](ACTIVE.md)
before claiming.
