# Task 464 — verify NoChainSync account queries

Component: **AccountManager NoChainSync query integration tests**.

Type: **Test**.

Dependencies: **[Task 391][t391]**.

## Rationale

[Issue #1250][issue1250] requires AccountManager integration coverage. At
`sql-wallet` revision `48e7f4b`, [existing account itests][source] already
verify root-derived `NoChainSync=true` through `GetAccount` and reopen, and
reject unsupported kvdb creation. [Task 430][t430] owns ordinary accounts'
complete query results, including `NoChainSync=false`; its [PR #1389][pr1389]
is still open at this audit. Neither coverage set needs another matrix here.

The remaining evidence is true values through SQL list queries and Task 391's
future path-bound account identity through all public account views.

## Details

- Extend existing root-derived true-value fixtures independently of Task 391's
  path-bound shape. Add that shape after its production contract lands;
  completion of the full task still requires 391.
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
  policy before and after reopen. This new shape has no existing query proof.
- Modern kvdb cannot represent these persisted SQL-only shapes. Document the
  structural exclusion at registration, without a skip in a shared case or
  another unsupported-creation test; existing mutation tests own that error.
- Do not repeat ordinary-false queries, creation rejections, or the general
  filter, ordering, exported-field, and reopen matrices owned by Tasks 429/430.
- Extend the existing `itest/account_manager_test.go` suite. Add no arbitrary
  helper, production file, or standalone test file; justify any unavoidable
  new file against the repository's current layout in the implementation PR.
- Document only non-obvious test contracts, rationales, invariants, and
  consequences. Keep Arrange, Act, and Assert structure clear without
  mandatory section comments that restate the test code.

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
[t391]: 391-path-bound-watch-only-xpub-accounts.md
[t430]: 430-account-query-itests.md
[source]: https://github.com/btcsuite/btcwallet/blob/48e7f4b7e0f3808b5022fd817d12c6a4361bb9c1/itest/account_manager_test.go
[pr1389]: https://github.com/btcsuite/btcwallet/pull/1389
