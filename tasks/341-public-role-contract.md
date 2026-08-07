# Task 341 — define semantic account identity

Component: **wallet-owned account identity types**.

Dependencies: **[Task 360][t360]**.

## Rationale

Public derivation and genesis validation need one backend-neutral account
selector plus optional account and fingerprint identity. SQL row IDs and kvdb
adapter account numbers are not portable public identity, and plain `uint32`
cannot distinguish an absent value from present-zero.

## Details

- Define a wallet-owned semantic account selector containing a key scope and
  exactly one of account name or optional BIP44 account number.
- Reject neither-set and both-set selectors before Store access.
- Never accept or expose a SQL account row ID, kvdb adapter ID, or recovery
  map key as a public selector.
- Define wallet-owned `AccountNumber` and `MasterFingerprint` value types.
  Represent their optional forms with pointers: nil is absent and a non-nil
  zero is present-zero. Do not add parallel `Has...` booleans.
- Follow Task 360's accepted distinction between internal scope-local kvdb
  adapter identity and portable semantic identity. Do not reopen that ADR.
- Keep AccountManager DTO conversion in Task 377, public error
  translation in the error tasks, and recovery-state implementation in
  Task 381.
- Do not add a generic row-ID selector or redesign unrelated public roles.

## Suggested fix

Add the smallest wallet-owned named values and an exactly-one selector with a
single validation path. Keep backend conversion private. Make the selector the
shared input consumed by Task 337, and make the optionals the shared facts
consumed by the AccountInfo DTO and Task 365.

## Verification

- An external-package test constructs valid name and numbered selectors and
  rejects both invalid shapes without Store access.
- External code can distinguish absent from present-zero account numbers and
  fingerprints using only wallet package types.
- Compile-time inspection proves the selector contains no backend identifier.

## Definition of done

- Wallet exports one semantic account selector and named optional identity
  values with preserved nil/zero semantics.
- Task 337 and Task 365 can consume those types without importing internal DB
  packages.
- No DTO conversion, backend error translation, recovery implementation, or
  backend ID is included.

[t360]: 360-document-kvdb-adapter-identity.md
