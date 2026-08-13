# Task 430 — cover public account query contracts

Component: **AccountManager query integration tests**.

Type: **Test**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] assigns Gustavo the AccountManager integration-test
lane. `ListAccounts`, `ListAccountsByScope`, `ListAccountsByName`, and
`GetAccount` expose the same public account view, but no focused real-backend
suite proves their filtering, missing-account, ordering, lifecycle, and reopen
contracts together across kvdb, SQLite, and PostgreSQL.

## Details

- Exercise all four public account query methods through real kvdb, SQLite,
  and PostgreSQL Wallets with deterministic default, created, and imported
  accounts.
- Register one shared public contract for all three database backends. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover empty filters, exact and absent filters, multiple scopes and names,
  stopped-wallet access, unknown identities, stable ordering where the public
  contract promises it, and agreement between list and point results.
- Reopen the same wallet and prove the complete public query view is unchanged.
- Exclude account mutation implementation, future wallet-owned DTO redesign,
  configurable identity semantics, addresses, direct Store access, RPC, and
  unrelated production changes.

## Suggested fix

Add one table-driven AccountManager query itest source that creates its fixture
through public mutation APIs, derives expected identities from returned public
objects, and compares filtered and point results without reading database rows.

## Verification

- Run the focused AccountManager query selection and fixture checks on kvdb,
  SQLite, and PostgreSQL.
- Verify results and ordering before and after a full reopen and ensure every
  case releases Wallet, Manager, miner, chain, and backend resources.

## Definition of done

- Every maintained public account query has representative success, absence,
  filter, lifecycle, and cross-method consistency coverage.
- The public account view remains stable after durable reopen.
- Shared cases use real kvdb, SQLite, and PostgreSQL Wallets without Store
  assertions or backend skip branches.
- No new query contract is introduced.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
