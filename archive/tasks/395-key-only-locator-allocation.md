# Task 395 — add AllocateNextKey for durable key allocation

Component: **AddressManager key allocation and SQL branch-counter mutation**.

Type: **Gap fix**.

Dependencies: **[Task 341][t341], [Task 344][t344], [Task 390][t390],
[Task 393][t393], [Task 399][t399], [Task 452][t452], and
[Task 467][t467]**.

## Rationale

Lnd `DeriveNextKey` needs a durable, unique account child. Reading a counter
and deriving without a mutation can return the same locator concurrently,
while the normal receiving path establishes live watches that a
`NoChainSync` account explicitly disables.

## Details

- Add `AllocateNextKey` to the existing public `AddressManager`. Accept a
  context, `AccountSelector`, and internal-branch boolean; return the public
  key and existing `AddressDerivation` metadata. Resolve the selected account
  and its schema, including custom purpose-1017 scopes, rather than inferring
  a standard scope from a receiving-address type.
- Require the selected account's persisted `NoChainSync=true` policy. Return
  `ErrAccountOperationUnsupported` before mutation for ordinary accounts and
  modern kvdb, which cannot represent that policy. Do not add a per-call
  policy override or change existing kvdb derivation methods.
- Atomically allocate exactly one external or internal child. Every successful
  call consumes a fresh index, even if earlier children have never appeared
  on chain. Return its public key, scope, account, branch, index, and master
  key fingerprint; never search for an unused child.
- Keep `NewAddress` and `NewBulkAddresses` as receiving operations that
  reject `NoChainSync`. Task 396 independently changes the receiving selector
  and oldest-unused semantics and removes `GetUnusedAddress`. Do not require
  that method to exist or restore it after removal. Task 396 is not a
  prerequisite for this API.
- AccountManager owns account creation and policy. Reuse Signer's existing
  `DerivePubKey` for a known account, branch, and index without advancing a
  counter. An lnd adapter can map `DeriveNextKey(family)` to `AllocateNextKey`
  and `DeriveKey(locator)` to `DerivePubKey`; add no second derivation wrapper,
  lnd-specific family type, implicit account creation, or lnd code here.
- Persist counter progress and every accepted child so concurrent and
  restarted calls cannot return the same locator. Consume invalid or colliding
  children and persist terminal exhaustion.
- Never retry an ambiguous commit in a way that can reuse a child. Return the
  shared indeterminate-commit error unless exact persisted facts prove the
  result.
- Create no live-watch obligation, chain notification, recovery target, or
  receiving-address promise for this operation.
- Persist the accepted child so ordinary address lookup, transaction ownership,
  and later signing continue to work. `NoChainSync` excludes automatic chain
  scanning, not ownership of a transaction supplied to the wallet.
- Reuse existing account counters, managed-address rows, and wallet-owned
  public errors. Do not add a locator table, reservation service, or another
  ownership model.
- Task 465 owns root-derived account integration assertions across real
  backends. Task 391 owns adapting this API and its focused tests to the
  path-bound supplied-XPub account shape introduced by that task.

## Suggested fix

Reuse the existing Store `NewDerivedAddress` mechanism, account counters, and
managed-address persistence behind AddressManager. Reuse Task 344's durable
allocation handling without calling its watched public batch API. Keep this
operation independent of receiving-address selection; introduce no new
manager, result wrapper, worker, table, or reservation mechanism.

## Verification

- Concurrent SQLite and PostgreSQL calls return distinct durable paths and
  continue doing so after reopen.
- Invalid children, script collisions, exhaustion, and ambiguous outcomes
  never reuse an index or relabel an existing owner.
- The result contains complete public metadata and remains resolvable through
  ordinary lookup, ownership, and signing paths.
- Prove allocation performs no watch persistence, recovery derivation, or
  chain call. Ordinary-account and modern-kvdb requests fail before mutation.
- Extend existing relevant unit tests and cases for this contract; Task 465
  owns public integration coverage. Do not duplicate existing counter tests.
- Include focused tests in this implementation PR without waiting for Tasks
  431 or 465. If relevant itests already exist, update behavior-dependent
  assertions in this PR; do not absorb their separate coverage matrices.

## Definition of done

`AddressManager.AllocateNextKey` allocates one durable unique child for a
`NoChainSync` account with complete key metadata and no chain-synchronization
side effects. Receiving APIs remain guarded, and known-index derivation keeps
using the existing `DerivePubKey` contract.

[t341]: ../archive/tasks/341-public-role-contract.md
[t344]: ../archive/tasks/344-bulk-address-allocation.md
[t390]: ../archive/tasks/390-custom-scope-account-creation.md
[t393]: ../archive/tasks/393-key-only-receiving-rejection.md
[t396]: 396-oldest-unused-address.md
[t399]: ../archive/tasks/399-zero-lookahead-persisted-targets.md
[t452]: ../archive/tasks/452-key-only-recovery-derivation.md
[t465]: 465-new-address-result-itests.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
