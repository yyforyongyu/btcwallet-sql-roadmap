# Task 353 — retire kvdb from the modern wallet API

Component: **Manager backend selection**.

Dependencies: **[Task 376][t376]**, plus production conversion evidence, an
identified completed consumer-cutover artifact or release proving maintained
consumers and deployed wallets no longer select kvdb, and an explicit
retirement decision.

## Rationale

After supported wallets can be converted to SQL, the maintained wallet library
should not continue carrying a second legacy Manager backend and its public
passphrase configuration.

## Details

- Require verified [Task 376][t376] conversion, identified completed consumer-
  cutover evidence or a release proving maintained consumers and deployed
  wallets no longer select kvdb, and an explicit retirement decision before
  scheduling. Conversion readiness alone is not cutover evidence.
- At scheduling time, enumerate and remove every implemented or pending
  modern-kvdb clause in the canonical backlog. Do not implement temporary
  behavior only to delete it. This includes, but is not limited to,
  [Task 330][t330], [Task 331][t331], [Task 343][t343], [Task 344][t344],
  [Task 352][t352], and the modern-kvdb clauses of
  [Task 382][t382] through [Task 385][t385].
  It also includes [Task 361][t361], [Task 362][t362], and the modern-kvdb
  clause of [Task 364][t364]. [Task 352][t352], [Task 361][t361], and
  [Task 362][t362] are archived entirely if retirement lands first; other
  tasks retain only their SQL or backend-neutral clauses. Preserve
  [Task 354][t354]'s accepted design record, and narrow [Task 386][t386] to
  cleanup for the surviving SQL consumers.
- Remove kvdb from the modern `NewManager` backend selection and configuration.
- Remove modern kvdb adapter and Vault wiring that has no remaining caller.
- Retire kvdb-only deprecated passphrase fields from modern public configs.
- Leave the deprecated Loader, RPC wallet, waddrmgr, wtxmgr, and their on-disk
  compatibility untouched unless separately approved.

## Suggested fix

Delete the modern Manager's kvdb backend case and its internal adapter wiring,
then simplify the unified configs around SQL ownership. Keep deprecated wallet
construction compiling against its existing walletdb path. Revalidate and
reduce every remaining task after removal; do not carry obsolete kvdb
prerequisites or acceptance criteria forward. Record the task-by-task removals
in the retirement PR description, not a runtime manifest.

## Verification

- Run focused Manager backend-selection tests proving kvdb is rejected and
  converted SQL wallets still create, load, close, and reopen.
- Build the deprecated wallet, Loader, waddrmgr, and wtxmgr paths without
  changing their behavior or disk compatibility.
- Audit modern configuration, adapter, Vault, and canonical-task references
  against the retirement ledger so no implemented or pending kvdb clause
  remains.

## Definition of done

- The modern Manager, Vault, adapters, and public configuration contain no
  kvdb backend path after verified conversion, completed consumer-cutover
  evidence, and the explicit retirement decision.
- Every canonical task's implemented or pending modern-kvdb clause is removed
  or archived according to the retirement ledger.
- Deprecated Loader, RPC wallet, waddrmgr, wtxmgr, and on-disk compatibility
  remain unchanged.

[t330]: 330-duplicate-wallet-create.md
[t331]: 331-manager-wallet-lifecycle.md
[t343]: 343-exact-account-creation.md
[t344]: 344-bulk-address-allocation.md
[t352]: 352-kvdb-combined-passphrase-rotation.md
[t354]: 354-vault-owned-secret-resolution.md
[t361]: 361-kvdb-imported-xpub-derivation.md
[t362]: 362-kvdb-definitive-passphrase-result.md
[t364]: 364-durable-watch-registration.md
[t376]: 376-orchestrate-legacy-conversion.md
[t382]: 382-vault-account-derivation.md
[t385]: 385-unsafe-signer-key-extraction.md
[t386]: 386-remove-raw-wallet-secret-seams.md
