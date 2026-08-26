# Task 429 — cover public account mutation contracts

Component: **AccountManager mutation integration tests**.

Type: **Test**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] assigns Gustavo the AccountManager integration-test
lane. `NewAccount`, `RenameAccount`, and `ImportAccount` mutate one durable
account namespace, so they form a coherent public contract slice whose commit
and reload behavior should be proven across every maintained database.

## Details

- Exercise `NewAccount`, `RenameAccount`, and `ImportAccount` only through a
  real Store-backed Wallet's public AccountManager.
- Register the same public contract for kvdb, SQLite, and PostgreSQL. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover valid mutation, duplicate name or identity, invalid name or input,
  unknown account, locked or watch-only restrictions where the maintained API
  defines them, and stopped-wallet admission.
- Assert every exported field of the wallet-owned `AccountInfo` from
  [Task 377][t377] returned by mutation and its public `GetAccount`
  postcondition. Cover the absent, present-zero, and present-nonzero optional
  identities from [Task 341][t341], plus counts, balances, schema, public key,
  fingerprint, and creation time. Never assert `AccountID` or another backend
  routing field.
- Reopen after each successful mutation and verify the complete exported
  account result remains durable.
- Keep current public entry-point semantics. Do not anticipate exact-number,
  custom-scope, path-bound, discovery-mode, or reconciliation behavior owned by
  Tasks 343 and 387–392.
- Exclude account queries beyond the narrow postcondition checks, addresses,
  direct Store calls, RPC, and unrelated production changes.

## Suggested fix

Add one mutation-focused AccountManager itest source with reusable account
fixtures and a durable reopen helper. Establish postconditions through
`GetAccount` rather than database rows, keep imported key material deterministic
and test-scoped, and keep the public result assertion helper inside the itest
package rather than adding a `package wallet_test` file.

## Verification

- Run the focused AccountManager mutation cases on kvdb, SQLite, and
  PostgreSQL.
- Verify every successful mutation survives a fresh Manager and Wallet, every
  rejection leaves the public account set unchanged, and all resources stop.
- Compare all exported AccountInfo fields through the imported `wallet` package
  on every backend; exported-value coverage does not use a separate
  `package wallet_test` source.

## Definition of done

- All three mutation APIs have representative success, stable-error,
  lifecycle-gate, and durable-reload coverage.
- Rejected operations do not create, rename, or import a partial account.
- Tests use public APIs on real kvdb, SQLite, and PostgreSQL Wallets, with no
  backend skip branch in the shared contract.
- The task introduces no new account behavior.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t341]: ../archive/tasks/341-public-role-contract.md
[t377]: ../archive/tasks/377-account-manager-wallet-dto.md
