# Task 464 — verify NoChainSync account queries

Component: **AccountManager NoChainSync query integration tests**.

Type: **Test**.

Dependencies: **[Task 391][t391]**.

## Rationale

[Issue #1250][issue1250] requires AccountManager integration coverage. Task 430
owns baseline account-query coverage, not this task's fixtures. The new
`AccountInfo.NoChainSync` value and SQL-only account shapes need a separate
public integration-test outcome after their production contracts land.

## Details

- Keep ordinary-false and root-derived-true fixtures independent of Task
  391's path-bound shape. Add the path-bound case separately after its
  production contract lands; completion of this full task still requires 391.
- This final field/account-shape matrix remains Phase 2 verification, not a
  blanket prerequisite for legacy conversion, whose input has NoChainSync
  false. Do not skip a required case to declare this task complete.
- Reuse existing imported-`wallet` query fixtures without waiting for Task
  430. Preserve its baseline coverage if it has already landed.
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
- Document only non-obvious test contracts, rationales, invariants, and
  consequences. Keep Arrange, Act, and Assert structure clear without
  mandatory section comments that restate the test code.

## Suggested fix

Add one focused table to the existing AccountManager integration suite. Reuse
available public query helpers and add only ordinary-false, SQL-true, and
kvdb-unsupported fixtures; Task 430 need not land first.

## Verification

- Run the ordinary-false case unchanged on kvdb, SQLite, and PostgreSQL.
- Run root-derived and path-bound true-value cases on SQLite and PostgreSQL,
  including a durable reopen.
- Prove the kvdb true-value request fails before mutation with the canonical
  unsupported identity.

## Definition of done

- The new public account field has focused real-backend coverage without
  duplicating baseline account-query cases.
- SQL-only true values remain durable on both dialects, and kvdb's structural
  exclusion is explicit.
- No new query behavior, public API, Store assertion, arbitrary file, or
  duplicated Task 430 case is introduced.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t391]: 391-path-bound-watch-only-xpub-accounts.md
[t430]: 430-account-query-itests.md
