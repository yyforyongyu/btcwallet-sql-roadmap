# Recommended work queue

This file shows the current execution frontier. Individual task files own the
complete dependency contract; this queue intentionally does not reproduce the
full graph.

## Priority

1. Enable PostgreSQL for the existing component itests.
2. Finish the focused prerequisites for atomic wallet creation.
3. Make wallet and promised-account creation atomic.
4. Continue independent component, migration, itest, and benchmark work.

## Active critical path

- [Task 333](tasks/333-postgres-manager-harness.md) enables the existing itest
  harness on PostgreSQL in
  [btcwallet PR #1303](https://github.com/btcsuite/btcwallet/pull/1303).
- [Task 332](tasks/332-watch-only-script-policy.md) defines watch-only creation
  policy in
  [btcwallet PR #1305](https://github.com/btcsuite/btcwallet/pull/1305).

See [ACTIVE.md](ACTIVE.md) for ownership and coordination details.

## Ready in parallel

- [Task 359](tasks/359-revalidate-database-identity-design.md) — revalidate the
  durable database-identity design.
- [Task 360](tasks/360-document-kvdb-adapter-identity.md) — document the
  internal kvdb adapter identity in ADR 0013.

These tasks are dependency-free. They may be claimed by different developers
while Task 333 and Task 332 are active.

## Atomic-creation chain

After the corresponding predecessor completes:

1. [Task 340](tasks/340-durable-database-identity.md) follows Task 359, while
   [Task 341](tasks/341-public-role-contract.md) follows Task 360.
2. [Task 345](tasks/345-durable-watch-registration.md) follows Task 340, while
   [Task 337](tasks/337-imported-xpub-derivation.md) follows Task 341.
3. [Task 329](tasks/329-atomic-wallet-account-creation.md) follows Tasks 332,
   337, and 345.

Task 329 does not wait for modern-kvdb follow-ups or durable live-watch
delivery beyond the dependencies named in its task file.

## Independent work after atomic creation

The first independent candidates are:

- [Task 331](tasks/331-manager-wallet-lifecycle.md) — Manager and Wallet
  lifecycle concurrency.
- [Task 339](tasks/339-transaction-visibility.md) — public transaction
  visibility and status.
- [Task 350](tasks/350-native-pgx-postgres.md) — native pgx PostgreSQL storage.
- [Task 355](tasks/355-transaction-size-accounting.md) — transaction size
  accounting.
- [Task 357](tasks/357-transaction-amount-validation.md) — checked monetary
  aggregates.

Consult each task for its exact dependencies before claiming it. Remaining
work stays discoverable through the [active task catalog](tasks/README.md) and
enters this queue only when it approaches the execution frontier.
