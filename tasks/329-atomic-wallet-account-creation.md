# Task 329 — commit SQL wallet genesis atomically

Component: **Manager SQL genesis and SQL Store**.

Dependencies: **[Task 332][t332], [Task 337][t337], and
[Task 345][t345]**.

## Rationale

SQL wallet rows, encrypted secrets, and sync state are created in one
transaction, but default or requested initial accounts are added afterward.
A definite account-creation failure can therefore leave a wallet that does not
contain the accounts its supported create request promised.

## Details

- Cover the shared SQLite and PostgreSQL create path only.
- Treat [Task 341][t341] and [Task 340][t340] as transitive prerequisites
  through the direct dependencies; do not add duplicate direct edges.
- Prepare the supported request's ciphertext and public genesis facts before
  opening the SQL write transaction. Consume the accepted watch-only policy,
  wallet-owned selector, and persisted discovery mode from the dependencies;
  do not redesign them here.
- Extend one backend-local SQL wallet-create transaction to insert the wallet,
  encrypted wallet secrets, sync state, key scopes, default or numberless
  shell accounts, encrypted account secrets, immutable discovery modes, and
  checked `next_account_number` cursors.
- Create the canonical default accounts promised by a supported private-root
  request. Persist every default genesis account as `watch-chain`; reject a
  request that attempts a key-only genesis override before derivation,
  encryption, or SQL mutation. A scope containing canonical account zero
  starts its next-account cursor at one.
- Create exactly the numberless initial accounts promised by a supported shell
  request. Persist every shell genesis account as `watch-chain` and reject a
  key-only override at the same pre-mutation boundary. Those accounts do not
  advance a numbered account cursor.
- Keep derivation and encryption outside the SQL transaction. The transaction
  receives only prepared ciphertext and public persistence facts.
- Treat every definite insertion or commit failure as failure of the complete
  genesis operation. It must leave no wallet, secret, sync, scope, account, or
  cursor artifact.
- Assemble and publish the Wallet in `Manager.wallets` only after the complete
  transaction commits and runtime assembly succeeds. A definite genesis
  failure must leave no cached Wallet.
- Keep modern-kvdb creation parity outside this task. No maintained non-test
  shell-creation caller currently justifies temporary parity before
  [Task 353](353-retire-modern-kvdb-backend.md) retires that backend.
- Keep enhanced key-identity validation in
  [Task 365](365-validate-wallet-genesis-keys.md), ambiguous SQL commit
  reconciliation in [Task 366](366-reconcile-sql-genesis-commit.md), and
  secret-lifetime hardening in
  [Task 367](367-bound-sql-genesis-secrets.md).
- Keep duplicate-create policy in [Task 330][t330], public cancellation in
  [Task 358][t358], numbered root-derived account creation in
  [Task 343][t343], custom-scope creation in [Task 390][t390], and path-bound
  watch-only XPub creation in [Task 391][t391].
- Leave configurable key-only creation to Task 343's guarded runtime account
  path after its allocation, ownership, and recovery prerequisites complete;
  genesis does not expose that mode.
- Do not add a create manifest, replay state, generic write lock, Store
  callback, or cross-process coordination.

## Suggested fix

Give the SQL wallet-create operation a method-specific genesis payload and
extend its existing dialect-neutral transaction stages to persist the complete
wallet and promised-account state. Keep only dialect-specific query wiring in
the SQLite and PostgreSQL wrappers. Return the committed facts to Manager for
runtime assembly and cache publication after the transaction succeeds.

## Verification

- Shared transaction-stage tests inject a definite failure at each wallet,
  secret, sync, scope, account, account-secret, discovery-mode, and cursor
  boundary and prove complete rollback.
- The same Store conformance runs on SQLite and PostgreSQL and reopens the
  wallet to prove the promised account set, discovery modes, and cursors.
- Private-root fixtures prove each canonical scope contains account zero and
  advances its cursor to one; shell fixtures prove numberless initial accounts
  leave each numbered cursor unchanged. Reopen proves every default and shell
  genesis account is `watch-chain` on both dialects.
- Private-root and shell requests that attempt a key-only genesis mode fail
  before derivation, encryption, Store mutation, or cache publication and
  leave no wallet artifact.
- Manager fault injection proves no Wallet is cached after a definite backend
  failure and publication occurs only after committed genesis and successful
  runtime assembly.
- The PR adds no kvdb parity, reconciliation policy, key-validation policy,
  secret-lifetime cleanup, duplicate-create behavior, or contextual API.

## Definition of done

- One SQLite or PostgreSQL transaction commits every wallet and promised-
  account genesis fact for a supported request.
- A definite failure leaves no SQL artifact and publishes no Wallet.
- Reopened wallets expose the exact promised account set and checked scope
  cursors on both SQL dialects, with every genesis account fixed to
  `watch-chain`.
- Key-only mode is unavailable during genesis and remains confined to the
  guarded runtime account-creation path.
- The PR contains no modern-kvdb, manifest, replay, generic-lock, or unrelated
  public API work.

[t330]: 330-duplicate-wallet-create.md
[t332]: 332-watch-only-script-policy.md
[t337]: 337-imported-xpub-derivation.md
[t340]: 340-durable-database-identity.md
[t341]: ../archive/tasks/341-public-role-contract.md
[t343]: 343-exact-account-creation.md
[t345]: 345-durable-watch-registration.md
[t358]: 358-contextual-manager-operations.md
[t390]: 390-custom-scope-account-creation.md
[t391]: 391-path-bound-watch-only-xpub-accounts.md
