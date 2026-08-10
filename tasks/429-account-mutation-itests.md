# Task 429 — cover public account mutation contracts

Component: **AccountManager mutation integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] assigns Gustavo the AccountManager integration-test
lane. `NewAccount`, `RenameAccount`, and `ImportAccount` mutate one durable
account namespace, so they form a coherent public contract slice whose SQL
commit and reload behavior should be proven end to end.

## Details

- Exercise `NewAccount`, `RenameAccount`, and `ImportAccount` only through a
  real Store-backed Wallet's public AccountManager.
- Require SQLite and register the same cases for PostgreSQL execution after
  [Task 333][t333] enables the harness.
- Cover valid mutation, duplicate name or identity, invalid name or input,
  unknown account, locked or watch-only restrictions where the maintained API
  defines them, and stopped-wallet admission.
- Reopen after each successful mutation and verify the public account identity,
  name, scope, and imported identity remain durable.
- Keep current public entry-point semantics. Do not anticipate exact-number,
  custom-scope, path-bound, discovery-mode, or reconciliation behavior owned by
  Tasks 343 and 387–392.
- Exclude account queries beyond the narrow postcondition checks, addresses,
  direct Store calls, RPC, and production fixes.

## Suggested fix

Add one mutation-focused AccountManager itest source with reusable account
fixtures and a durable reopen helper. Establish postconditions through
`GetAccount` rather than SQL rows and keep imported key material deterministic
and test-scoped.

## Verification

- Run the focused SQLite AccountManager mutation case selection.
- After Task 333 merges, run the identical registration on PostgreSQL.
- Verify every successful mutation survives a fresh Manager and Wallet, every
  rejection leaves the public account set unchanged, and all resources stop.
- Verify the PR contains only integration-test or narrow harness changes.

## Definition of done

- All three mutation APIs have representative success, stable-error,
  lifecycle-gate, and durable-reload coverage.
- Rejected operations do not create, rename, or import a partial account.
- Tests use public APIs on a real SQLite Wallet and remain backend-neutral.
- The task introduces no new account behavior or production fix.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t333]: ../archive/tasks/333-postgres-manager-harness.md
