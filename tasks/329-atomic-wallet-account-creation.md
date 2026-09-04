# Task 329 — commit SQL wallet genesis atomically

Component: **Manager SQL genesis and SQL Store**.

Type: **Bug fix**.

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
  wallet-owned selector, and persisted `NoChainSync` state from dependencies;
  do not redesign them here.
- Extend one backend-local SQL wallet-create transaction to insert the wallet,
  encrypted wallet secrets, sync state, key scopes, default or numberless
  shell accounts, encrypted account secrets, immutable `NoChainSync` values,
  and checked `next_account_number` cursors.
- Create the canonical default accounts promised by a supported private-root
  request. The public Manager create request exposes no `NoChainSync` input;
  its private genesis planner assigns false to every default account. A scope
  containing canonical account zero starts its next-account cursor at one.
- Create exactly the numberless initial accounts promised by a supported shell
  request. The same private planner assigns false to every shell account, and
  those accounts do not advance a numbered account cursor.
- Treat the method-specific private genesis-account payload passed into the
  SQL transaction as the invariant boundary. Reject a malformed internal
  payload containing `NoChainSync=true` before any SQL mutation; do not invent
  a public override or a public rejection case for a field callers cannot set.
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
  [Task 365](365-validate-wallet-genesis-keys.md), ambiguous SQL commit
  reconciliation in [Task 366](366-reconcile-sql-genesis-commit.md), and
  secret-lifetime hardening in
  [Task 367](367-bound-sql-genesis-secrets.md).
- Keep duplicate-create policy in [Task 330][t330], public cancellation in
  [Task 358][t358], numbered root-derived account creation in
  [Task 343][t343], custom-scope creation in [Task 390][t390], and path-bound
  watch-only XPub creation in [Task 391][t391].
- Leave configurable `NoChainSync` creation to
  [Task 468](468-activate-no-chain-sync-account-creation.md)'s guarded runtime
  account path; genesis does not expose that property.
- Do not add a create manifest, replay state, generic write lock, Store
  callback, or cross-process coordination.
- Keep implementation local to existing Manager genesis, SQL Store, source
  files, and colocated tests. Add no parallel interface, arbitrary helper,
  unrelated production file, or standalone test file; justify any unavoidable
  new file against the repository's current layout in the implementation PR.
- Document non-obvious code blocks with the contract, rationale, invariant, or
  consequence they preserve; do not restate the code.
- Keep unit tests clearly structured as Arrange, Act, and Assert.
- Interface mocks, when needed, embed `mock.Mock`. Set `On(...).Return(...)`
  expectations during Arrange with the narrowest correct cardinality (`Once()`
  for one required call). During Assert, call `AssertExpectations(t)` directly;
  do not duplicate call assertions or wrap the result in `require.True`.

## Suggested fix

Give the SQL wallet-create operation a method-specific genesis payload and
extend its existing dialect-neutral transaction stages to persist the complete
wallet and promised-account state. Keep only dialect-specific query wiring in
the SQLite and PostgreSQL wrappers. Return the committed facts to Manager for
runtime assembly and cache publication after the transaction succeeds.

## Verification

- Shared transaction-stage tests inject a definite failure at each wallet,
  secret, sync, scope, account, account-secret, chain-sync-state, and cursor
  boundary and prove complete rollback.
- The same Store conformance runs on SQLite and PostgreSQL and reopens the
  wallet to prove the promised account set, `NoChainSync` values, and cursors.
- Private-root fixtures prove each canonical scope contains account zero and
  advances its cursor to one; shell fixtures prove numberless initial accounts
  leave each numbered cursor unchanged. Reopen proves every default and shell
  genesis account has `NoChainSync=false` on both dialects.
- Planner tests prove public private-root and shell requests materialize only
  false account facts. Store-boundary tests inject a malformed private
  genesis-account payload with true and prove rejection before SQL mutation
  or cache publication; no public request fixture fabricates that field.
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
  `NoChainSync=false`.
- `NoChainSync=true` is unavailable during genesis and remains confined to
  the guarded runtime account-creation path.
- The PR contains no modern-kvdb, manifest, replay, generic-lock, or unrelated
  public API work.

[t330]: 330-duplicate-wallet-create.md
[t332]: ../archive/tasks/332-watch-only-script-policy.md
[t337]: ../archive/tasks/337-imported-xpub-derivation.md
[t340]: ../archive/tasks/340-durable-database-identity.md
[t341]: ../archive/tasks/341-public-role-contract.md
[t343]: 343-exact-account-creation.md
[t345]: 345-durable-watch-registration.md
[t353]: ../archive/tasks/353-retire-modern-kvdb-backend.md
[t358]: 358-contextual-manager-operations.md
[t390]: 390-custom-scope-account-creation.md
[t391]: 391-path-bound-watch-only-xpub-accounts.md
