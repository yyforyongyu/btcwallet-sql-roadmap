# Task 464 — verify NoChainSync account queries

Component: **AccountManager NoChainSync query integration tests**.

Type: **Test**.

Dependencies: **[Task 391][t391]**.

## Rationale

[Issue #1250][issue1250] requires AccountManager integration coverage.
At `sql-wallet` revision `4236325cbb609881272f488e413b90ab4aa07fc6`,
[existing account itests][source] already verify root-derived
`NoChainSync=true` through `GetAccount` and reopen, and reject
unsupported kvdb creation. [Task 430][t430] owns ordinary accounts'
complete query results, including `NoChainSync=false`, through merged
[PR #1389][pr1389]. PR #1408 already proves declared imported account
identity and `GetAccount` across reopen. Neither coverage set needs
another matrix here.

The remaining evidence is true values through SQL list queries and Task
391's path-bound true values through the remaining public list views.

## Details

- Extend existing root-derived and declared-import true-value fixtures. Task
  391's production contract has landed; reuse `ImportAccountParams` with a
  declared scope and SQL `NoChainSync=true`, without a new creation shape.
- This final field/account-shape matrix remains Phase 2 verification, not a
  blanket prerequisite for legacy conversion, whose input has NoChainSync
  false. Do not skip a required case to declare this task complete.
- Reuse existing imported-`wallet` query fixtures without a hard dependency on
  Task 430's test completion. Preserve its ordinary three-backend assertions.
- On SQLite and PostgreSQL, extend root-derived true-value fixtures with
  `ListAccounts`, `ListAccountsByScope`, and `ListAccountsByName` before and
  after reopen. Compare the complete exported results against the same
  expected account; retain existing `GetAccount` assertions without duplication.
- For Task 391's path-bound accounts with `NoChainSync=true`, verify all three
  list methods and `GetAccount` preserve the complete exported identity and
  policy before and after reopen. Reuse PR #1408's `GetAccount`/reopen proof;
  add only missing true-value assertions and the three list views.
- Declared imports themselves work on kvdb, but persisted `NoChainSync=true`
  remains SQL-only; kvdb ignores that flag. Document this narrower
  structural exclusion at registration, without a skip in a shared case or
  another unsupported-creation test; existing mutation tests own that error.
- Do not repeat ordinary-false queries, creation rejections, or the general
  filter, ordering, exported-field, and reopen matrices owned by Tasks 429/430.
- Extend the existing `itest/account_manager_test.go` suite. Add no arbitrary
  helper, production file, or standalone test file; justify any unavoidable
  new file against the repository's current layout in the implementation PR.

## Suggested fix

Extend the existing exact/custom account cases with the missing list-view
assertions, then reuse the query helpers for path-bound identities. Do not
create another ordinary-account or kvdb-rejection suite.

## Verification

- Run the root-derived true-value list assertions and path-bound all-view
  assertions on SQLite and PostgreSQL, including a durable reopen.
- Preserve existing ordinary three-backend query and kvdb-rejection evidence;
  add no duplicate test registration for either contract.

## Definition of done

- Missing true-value list and path-bound identity assertions have focused
  real-backend coverage without duplicating baseline account-query cases.
- SQL-only true values remain durable on both dialects, and kvdb's structural
  exclusion is explicit.
- No new query behavior, public API, Store assertion, arbitrary file, or
  duplicated Task 430 case is introduced.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t391]: ../archive/tasks/391-path-bound-watch-only-xpub-accounts.md
[t430]: ../archive/tasks/430-account-query-itests.md
[source]: https://github.com/btcsuite/btcwallet/blob/4236325cbb609881272f488e413b90ab4aa07fc6/itest/account_manager_test.go
[pr1389]: https://github.com/btcsuite/btcwallet/pull/1389
