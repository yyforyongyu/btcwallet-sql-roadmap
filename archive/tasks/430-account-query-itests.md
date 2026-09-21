# Task 430 — cover public account query contracts

Component: **AccountManager query integration tests**.

Type: **Test**.

Dependencies: **[Task 378][t378]**.

## Rationale

[Issue #1250][issue1250] assigns Gustavo the AccountManager integration-test
lane. `ListAccounts`, `ListAccountsByScope`, `ListAccountsByName`, and
`GetAccount` expose the same public account view, but no focused real-backend
suite proves their filtering, missing-account, ordering, and reopen
contracts together across kvdb, SQLite, and PostgreSQL.

## Details

- Exercise all four public account query methods through real kvdb, SQLite,
  and PostgreSQL Wallets with deterministic default, created, and imported
  accounts.
- Register one shared public contract for all three database backends. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover empty filters, exact and absent filters, multiple scopes and names,
  unknown identities, stable ordering where the public contract promises it,
  and agreement between list and point results. An absent `GetAccount` matches
  `ErrAccountNotFound` through `errors.Is`, while list misses return an empty
  result and nil error without exposing a Store or legacy-manager sentinel.
- Compare every exported field of the wallet-owned `AccountInfo` from
  [Task 377][t377]. Cover the absent, present-zero, and present-nonzero optional
  identities from [Task 341][t341], plus counts, balances, schema, public key,
  fingerprint, and creation time. Never assert `AccountID` or another backend
  routing field.
- Reopen the same wallet and prove the complete public query view is unchanged.
- Exclude account mutation implementation, configurable identity semantics,
  addresses, direct Store access, RPC, and unrelated production changes.
- Keep implementation local to the existing AccountManager integration-test
  file and its repository-conventional helpers. Add no arbitrary helper,
  production file, or standalone test file; justify any unavoidable new file
  against the repository's current layout in the implementation PR.
- Document only non-obvious test contracts, rationales, invariants, and
  consequences. Keep Arrange, Act, and Assert structure clear without
  mandatory section comments that restate the test code.

## Suggested fix

Add one table-driven AccountManager query itest source that creates its fixture
through public mutation APIs, derives expected identities from returned public
objects, and compares filtered and point results without reading database rows.
Keep the public result assertion helper inside the itest package rather than
adding a `package wallet_test` file.

## Verification

- Run the focused AccountManager query selection and fixture checks on kvdb,
  SQLite, and PostgreSQL.
- Verify results and ordering before and after a full reopen and ensure every
  case releases Wallet, Manager, miner, chain, and backend resources.
- Compare all exported AccountInfo fields through the imported `wallet` package
  on every backend.

## Definition of done

- Every maintained public account query has representative success, absence,
  filter, and cross-method consistency coverage.
- The public account view remains stable after durable reopen.
- Shared cases use real kvdb, SQLite, and PostgreSQL Wallets without Store
  assertions or backend skip branches.
- No new query contract is introduced.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t341]: ../archive/tasks/341-public-role-contract.md
[t377]: ../archive/tasks/377-account-manager-wallet-dto.md
[t378]: ../archive/tasks/378-account-manager-error-boundary.md
