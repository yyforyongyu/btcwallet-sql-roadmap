# Task 378 — translate AccountManager backend errors

Component: **AccountManager public error boundary**.

Type: **Bug fix**.

Dependencies: **None**.

## Rationale

AccountManager currently returns `db.ErrAccountNotFound` and unwraps
`waddrmgr.ManagerError` directly. External callers therefore depend on a Store
or legacy-manager error contract that is not owned by wallet.

## Details

- Reuse the existing wallet-owned `ErrAccountNotFound` for account misses from
  SQL Store errors and legacy `waddrmgr.ErrAccountNotFound` manager errors.
- Inventory only the documented AccountManager outcomes of `NewAccount`,
  `GetAccount`, `RenameAccount`, and `ImportAccount`. Give any additional
  documented semantic condition one wallet-owned sentinel; do not export an
  internal error table wholesale.
- Preserve useful operation and selector context while wrapping only the
  wallet-owned sentinel with `%w`.
- Do not return `db` sentinels or `waddrmgr.ManagerError` values from the public
  methods. Unexpected internal failures may retain diagnostic context but must
  not make an internal sentinel part of the public `errors.Is` contract.
- Keep duplicate wallet creation in Task 330, DTO conversion in Task 377,
  exact root-derived validation in [Task 343][t343], and custom-scope schema
  validation in [Task 390][t390].

## Suggested fix

Add a small AccountManager-specific translation function keyed only by stable,
documented account outcomes. Call it once at each Store or legacy-manager exit
and remove the current `errors.As(..., waddrmgr.ManagerError)` passthrough.

## Verification

- External-package SQL and modern-kvdb tests prove account misses satisfy
  `errors.Is(err, wallet.ErrAccountNotFound)` and do not satisfy internal DB or
  legacy-manager error checks.
- One test covers each additional documented mapped condition, if any.
- An injected unexpected backend error is not accidentally promoted to a
  stable public sentinel.

## Definition of done

- AccountManager exposes only wallet-owned stable account errors.
- SQL and kvdb equivalent conditions produce the same `errors.Is` result.
- No DTO, Manager, AddressManager, or recovery contract changes are included.

[t343]: 343-exact-account-creation.md
[t390]: 390-custom-scope-account-creation.md
