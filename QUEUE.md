# Task dependency phases

The roadmap currently contains **39 active tasks**. No task in the current
Task 329-and-later series has been completed and archived yet; older archived
files are historical or superseded plans and do not count as completed tasks.

Every active task appears exactly once below. Phase 1 contains tasks with no
active-task prerequisite. Later phases are derived from dependency depth:
tasks in the same phase do not depend on one another and can proceed in
parallel.

Phases are not global barriers. A task can start as soon as its own
dependencies and gates are complete; it does not wait for unrelated tasks in
earlier phases. Individual task files remain authoritative for scope and
dependencies. [ACTIVE.md](ACTIVE.md) separately owns assignees, current work,
branches, PRs, and coordination; this file does not duplicate that live state.

## Phase 1 — no active-task prerequisites

| Task | Outcome | Requires |
| --- | --- | --- |
| [Task 332][t332] | Define watch-only policy | — |
| [Task 333][t333] | Add the PostgreSQL harness | — |
| [Task 331][t331] | Harden Manager lifecycle | — |
| [Task 350][t350] | Use native pgx | — |
| [Task 339][t339] | Define transaction visibility | — |
| [Task 352][t352] | Make kvdb passphrase rotation atomic | — |
| [Task 355][t355] | Correct transaction-size accounting | — |
| [Task 357][t357] | Validate transaction amounts | — |
| [Task 359][t359] | Revalidate database identity design | — |
| [Task 360][t360] | Document kvdb adapter identity | — |

## Phase 2

| Task | Outcome | Requires |
| --- | --- | --- |
| [Task 330][t330] | Reject duplicate creation | 331, 333 |
| [Task 338][t338] | Return a definitive passphrase result | 331 |
| [Task 340][t340] | Persist database identity | 359 |
| [Task 341][t341] | Make public role contracts wallet-owned | 360 |
| [Task 346][t346] | Preserve PSBT funding metadata | 357 |
| [Task 354][t354] | Move secret resolution behind the Vault | 352; ADR 0010 |
| [Task 356][t356] | Defer change allocation until viable | 355, 357 |

Task 354 waits for Task 352 while modern kvdb remains. If kvdb retires first,
it waits for [Task 353][t353] instead and moves to a later phase. Task 356 also
waits for the required nested-module changes from Tasks 355 and 357 to be
released.

## Phase 3

| Task | Outcome | Requires |
| --- | --- | --- |
| [Task 334][t334] | Coordinate shared chain state | 333, 340; ADR 0002 |
| [Task 337][t337] | Restore imported-XPub derivation | 341 |
| [Task 342][t342] | Accept bounded recovery windows | 341 |
| [Task 345][t345] | Persist account discovery mode | 340 |
| [Task 362][t362] | Return a definitive kvdb passphrase result | 338, 352 |
| [Task 365][t365] | Validate genesis key identities | 332, 341 |

Task 334 requires an accepted ADR superseding ADR 0002.

## Phase 4

| Task | Outcome | Requires |
| --- | --- | --- |
| [Task 329][t329] | Commit SQL genesis atomically | 332, 337, 345 |
| [Task 335][t335] | Convert kvdb wallets to SQL | 332, 337, 340, 345 |
| [Task 336][t336] | Remove redundant Wallet tests | 338, 354, 362 |
| [Task 349][t349] | Expose transaction invalidation | 334, 339, 340; ADR 0006 |
| [Task 361][t361] | Support kvdb imported-XPub derivation | 337 |
| [Task 364][t364] | Deliver durable live watches | 345; ADRs 0004/0005 |

Task 336 uses the listed dependencies while modern kvdb remains. If kvdb
retires first, it waits for [Task 353][t353] instead and may move to a later
phase. Task 349 requires an accepted ADR 0006 amendment making normalized
inputs authoritative. Task 364 requires accepted ADR 0004 and ADR 0005
amendments for live-watch delivery.

## Phase 5

| Task | Outcome | Requires |
| --- | --- | --- |
| [Task 343][t343] | Support exact accounts | 332, 337, 340, 341, 345, 354, 364; ADRs 0012/0013 |
| [Task 344][t344] | Allocate addresses atomically | 337, 364 |
| [Task 348][t348] | Deliver watched change | 346, 355, 356, 364 |
| [Task 351][t351] | Enforce SQL UTXO integrity | 340, 349; ADR 0006 |
| [Task 353][t353] | Retire modern kvdb | 335; evidence; cutover |
| [Task 363][t363] | Bound recovery scan work | 337, 342, 345, 361, 364; ADR 0004 |
| [Task 366][t366] | Reconcile SQL genesis commits | 329 |
| [Task 367][t367] | Bound genesis secret lifetime | 329 |

Task 351 requires an accepted ADR 0006 amendment or supersession covering
declarative UTXO ownership. Task 353 also requires production conversion
evidence and an explicit consumer cutover decision. Task 363 waits for Task 361
while modern kvdb remains; if kvdb retires first, it waits for Task 353 instead
and may move to a later phase.

## Phase 6

| Task | Outcome | Requires |
| --- | --- | --- |
| [Task 347][t347] | Complete PSBT authorization | 343, 346, 354 |
| [Task 358][t358] | Make Manager operations cancellable | 331, 366, 367 |

## Maintaining the phases

When a task completes or its dependency contract changes, refresh its phase
and every downstream phase affected by that edge. Preserve one row per active
task. A phase never authorizes claiming a task before all requirements in its
task file are complete; consult the task and [ACTIVE.md](ACTIVE.md) first.

[t329]: tasks/329-atomic-wallet-account-creation.md
[t330]: tasks/330-duplicate-wallet-create.md
[t331]: tasks/331-manager-wallet-lifecycle.md
[t332]: tasks/332-watch-only-script-policy.md
[t333]: tasks/333-postgres-manager-harness.md
[t334]: tasks/334-shared-chain-state.md
[t335]: tasks/335-kvdb-to-sql-migration.md
[t336]: tasks/336-test-cleanup.md
[t337]: tasks/337-imported-xpub-derivation.md
[t338]: tasks/338-passphrase-result.md
[t339]: tasks/339-transaction-visibility.md
[t340]: tasks/340-durable-database-identity.md
[t341]: tasks/341-public-role-contract.md
[t342]: tasks/342-recovery-window-validation.md
[t343]: tasks/343-exact-account-creation.md
[t344]: tasks/344-bulk-address-allocation.md
[t345]: tasks/345-durable-watch-registration.md
[t346]: tasks/346-psbt-funding-metadata.md
[t347]: tasks/347-psbt-signing-authorization.md
[t348]: tasks/348-watched-change-delivery.md
[t349]: tasks/349-transaction-invalidation.md
[t350]: tasks/350-native-pgx-postgres.md
[t351]: tasks/351-sql-utxo-integrity.md
[t352]: tasks/352-kvdb-combined-passphrase-rotation.md
[t353]: tasks/353-retire-modern-kvdb-backend.md
[t354]: tasks/354-vault-owned-secret-resolution.md
[t355]: tasks/355-transaction-size-accounting.md
[t356]: tasks/356-defer-change-script-allocation.md
[t357]: tasks/357-transaction-amount-validation.md
[t358]: tasks/358-contextual-manager-operations.md
[t359]: tasks/359-revalidate-database-identity-design.md
[t360]: tasks/360-document-kvdb-adapter-identity.md
[t361]: tasks/361-kvdb-imported-xpub-derivation.md
[t362]: tasks/362-kvdb-definitive-passphrase-result.md
[t363]: tasks/363-recovery-scan-work-bound.md
[t364]: tasks/364-durable-watch-registration.md
[t365]: tasks/365-validate-wallet-genesis-keys.md
[t366]: tasks/366-reconcile-sql-genesis-commit.md
[t367]: tasks/367-bound-sql-genesis-secrets.md
