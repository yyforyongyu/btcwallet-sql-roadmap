# Task 464 — verify NoChainSync account queries

Component: **AccountManager NoChainSync query integration tests**.

Type: **Test**.

Dependencies: **[Task 391][t391] and [Task 430][t430]**.

## Rationale

[Issue #1250][issue1250] requires AccountManager integration coverage. Task 430
owns the existing account-query contract and must remain unchanged. The new
`AccountInfo.NoChainSync` value and SQL-only account shapes need a separate
public integration-test outcome after their production contracts land.

## Details

- Extend Task 430's imported-`wallet` query suite without changing that task's
  original cases or definition of done.
- On kvdb, SQLite, and PostgreSQL, prove ordinary accounts report
  `NoChainSync=false` through `ListAccounts`, `ListAccountsByScope`,
  `ListAccountsByName`, and `GetAccount` before and after reopen.
- On SQLite and PostgreSQL, create root-derived and path-bound accounts with
  `NoChainSync=true`, query them through every public account view, and require
  the complete exported `AccountInfo` result to retain the value after reopen.
- Modern kvdb cannot represent the persisted property. Keep it outside the
  SQL-only true-value cases and require a true creation request to match
  `ErrAccountOperationUnsupported` through `errors.Is` before mutation.
- Do not repeat Task 430's general filter, ordering, exported-field, or reopen
  matrix. Add only the new field, SQL account-shape, backend-rejection, and
  durability assertions listed here.
- Extend the existing `itest/account_manager_test.go` suite. Add no arbitrary
  helper, production file, or standalone test file; justify any unavoidable
  new file against the repository's current layout in the implementation PR.
- Document every added code block inline with why it exists and how it works;
  comments that merely restate the code do not satisfy this requirement. Give
  every test detailed Arrange, Act, and Assert comments explaining its setup,
  operation, and evidence.

## Suggested fix

Add one focused table to the existing AccountManager integration suite. Reuse
Task 430's public query helpers and add only ordinary-false, SQL-true, and
kvdb-unsupported fixtures.

## Verification

- Run the ordinary-false case unchanged on kvdb, SQLite, and PostgreSQL.
- Run root-derived and path-bound true-value cases on SQLite and PostgreSQL,
  including a durable reopen.
- Prove the kvdb true-value request fails before mutation with the canonical
  unsupported identity.

## Definition of done

- The new public account field has focused real-backend coverage without
  changing Task 430's existing contract.
- SQL-only true values remain durable on both dialects, and kvdb's structural
  exclusion is explicit.
- No new query behavior, public API, Store assertion, arbitrary file, or
  duplicated Task 430 case is introduced.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t391]: 391-path-bound-watch-only-xpub-accounts.md
[t430]: 430-account-query-itests.md
