# Task 336 — remove redundant Wallet tests

Component: **Controller and Wallet test suite**.

Dependencies: **[Task 338][t338] and [Task 354][t354], plus
[Task 362][t362] while modern kvdb remains or [Task 353][t353] retiring that
backend first**.

## Rationale

The delivered stack duplicates focused passphrase coverage at the Wallet layer.
The same behavior is already owned by focused keyvault and backend tests.

## Details

- Keep focused keyvault and backend tests as the behavior owners.
- Retain at most one small public Wallet smoke per backend.
- Retain [Task 338][t338]'s SQL acceptance-latched cancellation coverage and,
  while modern kvdb remains, [Task 362][t362]'s backend parity at the
  Controller layer. Remove only older interruption cases made redundant by
  those contracts.
- Remove runtime-cache secret pass-through tests only after [Task 354][t354] moves
  signer secret resolution behind the Vault.
- Add no daemon itest or behavior change.

## Suggested fix

Delete the duplicate Wallet passphrase matrix and interruption cases that add no
distinct assertion. After [Task 354][t354], remove tests and mocks for obsolete
`runtimeCache.GetAccountSecret` and `GetAddressSecret` methods.

## Verification

- Run the focused keyvault and backend passphrase tests plus
  [Task 338][t338]'s SQL Controller cancellation tests and, while modern kvdb
  remains, [Task 362][t362]'s parity tests after deleting the duplicate Wallet
  cases.
- After [Task 354][t354], run the signer and account-manager tests that replace
  obsolete runtime-cache pass-through mocks.
- Inspect the remaining Wallet tests and confirm each retained case asserts a
  distinct public smoke rather than duplicating narrower coverage.

## Definition of done

- Redundant Wallet-level passphrase and cache pass-through tests are removed
  only after their focused replacement coverage is stable.
- At most one public Wallet smoke remains per backend, and no behavior or
  daemon integration scenario changes.

[t338]: 338-passphrase-result.md
[t353]: 353-retire-modern-kvdb-backend.md
[t354]: 354-vault-owned-secret-resolution.md
[t362]: 362-kvdb-definitive-passphrase-result.md
