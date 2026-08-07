# Task 336 — remove redundant Wallet tests

Component: **Controller and Wallet test suite**.

Dependencies: **[Task 338][t338] and [Task 354][t354]**.

## Rationale

The delivered stack duplicates focused passphrase coverage at the Wallet layer.
The same behavior is already owned by focused keyvault and backend tests.

## Details

- Keep focused keyvault and backend tests as the behavior owners.
- Retain at most one small public Wallet smoke per backend.
- Retain [Task 338][t338]'s acceptance-latched cancellation coverage at the Controller
  layer; remove only older interruption cases made redundant by that contract.
- Remove runtime-cache secret pass-through tests only after [Task 354][t354] moves
  signer secret resolution behind the Vault.
- Add no daemon itest or behavior change.

## Potential fix

Delete the duplicate Wallet passphrase matrix and interruption cases that add no
distinct assertion. After [Task 354][t354], remove tests and mocks for obsolete
`runtimeCache.GetAccountSecret` and `GetAddressSecret` methods.

## Done when

Each behavior remains covered once at the narrowest useful layer, with no test
that exists only to exercise an obsolete cache pass-through.

[t338]: 338-passphrase-result.md
[t354]: 354-vault-owned-secret-resolution.md
