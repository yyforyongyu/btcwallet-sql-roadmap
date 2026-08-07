# Task 353 — retire kvdb from the modern wallet API

Component: **Manager backend selection**.

Dependencies: **[Task 335][t335]**, production conversion evidence, and an explicit
consumer-cutover decision.

## Rationale

After supported wallets can be converted to SQL, the maintained wallet library
should not continue carrying a second legacy Manager backend and its public
passphrase configuration.

## Details

- Require verified [Task 335][t335] conversion and an explicit consumer cutover
  decision before scheduling.
- At scheduling time, enumerate and remove every implemented or pending
  modern-kvdb clause in the canonical backlog. Do not implement temporary
  behavior only to delete it. This includes, but is not limited to,
  [Task 329][t329], [Task 330][t330], [Task 331][t331], [Task 337][t337],
  [Task 343][t343]–[Task 345][t345], [Task 352][t352], and [Task 354][t354].
  [Task 352][t352] is archived entirely if retirement lands first; other tasks
  retain only their SQL or backend-neutral clauses.
- Remove kvdb from the modern `NewManager` backend selection and configuration.
- Remove modern kvdb adapter and Vault wiring that has no remaining caller.
- Retire kvdb-only deprecated passphrase fields from modern public configs.
- Leave the deprecated Loader, RPC wallet, waddrmgr, wtxmgr, and their on-disk
  compatibility untouched unless separately approved.

## Potential fix

Delete the modern Manager's kvdb backend case and its internal adapter wiring,
then simplify the unified configs around SQL ownership. Keep deprecated wallet
construction compiling against its existing walletdb path. Revalidate and
reduce every remaining task after removal; do not carry obsolete kvdb
prerequisites or acceptance criteria forward. Record the task-by-task removals
in the retirement PR description, not a runtime manifest.

## Done when

The maintained wallet library exposes only SQL Manager backends, migrated
consumers open their converted wallets, and the deprecated wallet still builds
without gaining new behavior.

[t329]: 329-atomic-wallet-account-creation.md
[t330]: 330-duplicate-wallet-create.md
[t331]: 331-manager-wallet-lifecycle.md
[t335]: 335-kvdb-to-sql-migration.md
[t337]: 337-imported-xpub-derivation.md
[t343]: 343-exact-account-creation.md
[t345]: 345-durable-watch-registration.md
[t352]: 352-kvdb-combined-passphrase-rotation.md
[t354]: 354-vault-owned-secret-resolution.md
