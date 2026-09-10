# Task 378 — translate AccountManager backend errors

Component: **AccountManager public error boundary**.

Type: **Bug fix**.

Dependencies: **None**.

## Rationale

AccountManager returns `db` sentinels and `waddrmgr.ManagerError` values from
maintained public methods. An lnd adapter would therefore need backend-specific
error handling for the same account outcome, and changing or retiring a Store
would change its public contract.

The wallet package must own the small set of stable errors callers use to
distinguish absence, duplication, invalid input, unsupported operations,
locked state, and derivation exhaustion.

## Details

- Reuse existing wallet-owned `ErrAccountNotFound`, `ErrInvalidAccountKey`,
  `ErrInvalidParam`, and `ErrStateForbidden` identities.
- Define only `ErrAccountAlreadyExists`,
  `ErrAccountOperationUnsupported`, and
  `ErrAccountDerivationExhausted` for currently reachable account outcomes.
- Make absent query or rename targets match `ErrAccountNotFound`; duplicate
  names, rename targets, and self-renames match
  `ErrAccountAlreadyExists`; invalid requests and keys retain their existing
  wallet identities.
- Make watch-only derivation, unsupported spendable-SQL import, and terminal
  account exhaustion match the appropriate wallet-owned identity without
  exposing their backend errors.
- Preserve context cancellation and deadlines. A locked or concurrently
  locking Wallet returns `ErrStateForbidden` without leaking Vault errors.
- Preserve useful operation context with `%w`, but unexpected internal errors
  must not accidentally match another stable sentinel.
- Keep exact-number, custom-scope, XPub-collision, and future bounded-work
  behavior in their owning tasks. Tasks 430 and 462 own broad public
  integration coverage.
- Do not introduce Manager, AddressManager, recovery, or Store behavior.

## Suggested fix

Translate errors once at each AccountManager public exit using a small local
mapping. The helper structure and lookup order are flexible as long as every
documented public outcome has the same identity on SQL and kvdb.

## Verification

- Exercise every listed outcome through maintained AccountManager methods with
  backend, Vault, and lifecycle errors injected at their reachable boundaries.
- Prove public errors match only their wallet-owned identity and no `db`,
  `waddrmgr`, or Vault sentinel escapes.
- Verify validation and occupied-name precedence occur before mutation and an
  empty list query remains an empty result with no error.
- Leave full three-backend query and mutation scenarios to Tasks 430 and 462.

## Definition of done

Currently supported AccountManager outcomes have one backend-neutral
wallet-owned `errors.Is` contract, with no leaked Store, legacy-manager, or
Vault error identity and no adjacent account behavior added.

[t343]: 343-exact-account-creation.md
[t388]: 388-runtime-account-xpub-collisions.md
[t390]: 390-custom-scope-account-creation.md
[t430]: 430-account-query-itests.md
[t460]: 460-sequential-account-invalid-children.md
[t462]: 462-account-mutation-error-itests.md
