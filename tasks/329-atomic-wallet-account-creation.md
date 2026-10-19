# Task 329 — commit SQL wallet genesis atomically

Component: **Manager SQL genesis and SQL Store**.

Type: **Bug fix**.

Dependencies: **[Task 332][t332], [Task 337][t337], [Task 345][t345],
[Task 391][t391], and [Task 481][t481]**.

## Rationale

SQL wallet rows, encrypted secrets, and sync state are created in one
transaction, but default or requested initial accounts are added afterward.
At `sql-wallet` revision
`4236325cbb609881272f488e413b90ab4aa07fc6`, `Manager.Create` still calls
`importInitialAccounts` after `createWallet` returns, after SQL
`CreateWalletWithOps` has committed.
A definite account-creation failure can therefore leave a wallet that does not
contain the accounts its supported create request promised.

## Details

- Cover the shared SQLite and PostgreSQL create path only.
- Prepare the supported request's ciphertext and public genesis facts before
  opening the SQL write transaction. Consume the accepted watch-only policy,
  wallet-owned selector, and persisted `NoChainSync` state from dependencies;
  do not redesign them here.
- Extend one backend-local SQL wallet-create transaction to insert the wallet,
  encrypted wallet secrets, sync state, key scopes, default or requested
  shell accounts, encrypted account secrets, immutable `NoChainSync` values,
  and checked `next_account_number` cursors.
- Create the canonical default accounts promised by a supported private-root
  request with `NoChainSync=false`. A scope containing canonical account zero
  starts its next-account cursor at one.
- Preserve the current shared `ImportAccountParams` semantics for shell
  initial accounts, including declared path-bound numbers, fingerprints,
  per-account schemas, and SQL `NoChainSync`. Numberless imports do not advance
  a numbered cursor; declared numbers advance it under the existing rule.
  PRs #1408 and #1409 already provide these shapes; atomic genesis must not
  narrow them or add another public import input.
- Keep derivation and encryption outside the SQL transaction. The transaction
  receives only prepared ciphertext and public persistence facts.
- Treat every definite insertion or commit failure as failure of the complete
  genesis operation. It must leave no wallet, secret, sync, scope, account, or
  cursor artifact.
- Assemble and publish the Wallet in `Manager.wallets` only after the complete
  transaction commits and runtime assembly succeeds. A definite genesis
  failure must leave no cached Wallet.
- Keep modern-kvdb creation parity outside this SQL genesis task. Preserve its
  maintained behavior; deferred [Task 353][t353] does not authorize its removal.
- Keep enhanced key-identity validation in
  [Task 365](365-validate-wallet-genesis-keys.md) and secret-lifetime hardening
  in [Task 367](367-bound-sql-genesis-secrets.md).
- Lost commit confirmation must not publish a Wallet, trigger an automatic
  retry, or imply rollback. Task 358 owns the shared indeterminate-error
  mapping; add no genesis-plan readback or reconciliation.
- Keep duplicate-create policy in [Task 330][t330], public cancellation in
  [Task 358][t358], numbered root-derived account creation in
  [Task 343][t343], custom-scope creation in [Task 390][t390], and path-bound
  watch-only XPub creation in [Task 391][t391].
- Preserve [Task 468][t468]'s separate root-derived runtime account behavior;
  this task only makes the existing genesis request atomic.
- Do not add a create manifest, replay state, generic write lock, Store
  callback, or cross-process coordination.

## Suggested fix

Extend the existing SQL wallet-create transaction to persist the complete
wallet and promised-account state using prepared ciphertext and public facts.
Reuse `CreateWalletWithOps` and the current backend query wiring. Return
committed facts to Manager for runtime assembly and cache publication after
the transaction succeeds.

## Verification

- Inject a definite account-insertion failure after wallet insertion and prove
  the complete genesis rolls back on both SQL dialects.
- The same Store conformance runs on SQLite and PostgreSQL and reopens the
  wallet to prove the promised account set, `NoChainSync` values, and cursors.
- Private-root fixtures prove each canonical scope contains account zero and
  advances its cursor to one with `NoChainSync=false`. Shell fixtures cover
  both inferred numberless and declared numbered imports, per-account schema
  overrides, and both SQL `NoChainSync` values. Reopen preserves every accepted
  identity and policy; cursors follow the existing numbered/numberless rules.
- Manager fault injection proves no Wallet is cached after a definite backend
  failure and publication occurs only after committed genesis and successful
  runtime assembly.
- The PR adds no kvdb parity, reconciliation policy, key-validation policy,
  secret-lifetime cleanup, duplicate-create behavior, or contextual API.

## Definition of done

- One SQLite or PostgreSQL transaction commits every wallet and promised-
  account genesis fact for a supported request.
- A definite failure leaves no SQL artifact and publishes no Wallet.
- Reopened wallets expose the exact promised account set, schemas, policies,
  and checked scope cursors on both SQL dialects. Atomic genesis preserves the
  current accepted initial-import request shapes.
- The PR contains no modern-kvdb, manifest, replay, generic-lock, or unrelated
  public API work.

[t330]: 330-duplicate-wallet-create.md
[t332]: ../archive/tasks/332-watch-only-script-policy.md
[t337]: ../archive/tasks/337-imported-xpub-derivation.md
[t343]: ../archive/tasks/343-exact-account-creation.md
[t345]: ../archive/tasks/345-durable-watch-registration.md
[t353]: ../archive/tasks/353-retire-modern-kvdb-backend.md
[t358]: 358-contextual-manager-operations.md
[t390]: ../archive/tasks/390-custom-scope-account-creation.md
[t391]: ../archive/tasks/391-path-bound-watch-only-xpub-accounts.md
[t468]: ../archive/tasks/468-activate-no-chain-sync-account-creation.md
[t481]: ../archive/tasks/481-persist-imported-account-address-schema.md
