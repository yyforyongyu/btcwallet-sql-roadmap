# Task 430 — cover public account query contracts

Component: **AccountManager query integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] assigns Gustavo the AccountManager integration-test
lane. `ListAccounts`, `ListAccountsByScope`, `ListAccountsByName`, and
`GetAccount` expose the same public account view, but no focused real-SQL suite
proves their filtering, missing-account, ordering, lifecycle, and reopen
contracts together.

## Details

- Exercise all four public account query methods through a real SQL-backed
  Wallet with deterministic default, created, and imported accounts.
- Require SQLite and keep one backend-neutral registration for PostgreSQL after
  [Task 333][t333].
- Cover empty filters, exact and absent filters, multiple scopes and names,
  stopped-wallet access, unknown identities, stable ordering where the public
  contract promises it, and agreement between list and point results.
- Reopen the same wallet and prove the complete public query view is unchanged.
- Exclude account mutation implementation, future wallet-owned DTO redesign,
  configurable identity semantics, addresses, direct Store access, RPC, and
  production fixes.

## Suggested fix

Add one table-driven AccountManager query itest source that creates its fixture
through public mutation APIs, derives expected identities from returned public
objects, and compares filtered and point results without reading SQL rows.

## Verification

- Run the focused SQLite AccountManager query selection and its fixture checks.
- After Task 333 merges, run the same source on PostgreSQL.
- Verify results and ordering before and after a full reopen and ensure every
  case releases Wallet, Manager, miner, chain, and backend resources.
- Verify the PR remains test-only.

## Definition of done

- Every maintained public account query has representative success, absence,
  filter, lifecycle, and cross-method consistency coverage.
- The public account view remains stable after durable reopen.
- The backend-neutral cases use a real SQLite Wallet without Store assertions.
- No query contract or production behavior is changed by the PR.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t333]: 333-postgres-manager-harness.md
