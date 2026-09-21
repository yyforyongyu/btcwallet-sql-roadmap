# Task 462 — cover public account mutation errors

Component: **AccountManager mutation-error integration tests**.

Type: **Test**.

Dependencies: **[Task 391][t391]**.

## Rationale

[Issue #1250][issue1250] requires AccountManager integration coverage.
At `sql-wallet` revision `48e7f4b`, [existing account cases][source] already
cover ordinary, exact, and custom creation, including several stable rejection
identities. `testAccountManagerRejectInvalidImportKey` also checks nil/private
keys with `ErrorIs`, and stopped mutation calls already check their identity.
Rename, locked-create, and some ordinary-import cases still use generic
`require.Error`; self-rename, collision precedence, and supplied path-bound
XPub shapes remain missing. This task owns only those assertion upgrades and
uncovered final-contract cases.

## Details

- Upgrade the existing rename, locked-create, and ordinary-import rejection
  assertions in place. Reuse the nil/private-key, stopped-call, and ordinary
  creation cases already checking stable identities; add only missing shapes
  or precedence, not another success or query matrix.
- Use imported `wallet` mutation APIs and public account views on real kvdb,
  SQLite, and PostgreSQL. Share representable cases unchanged; register
  SQL-only exact, custom-scope, path-bound, and `NoChainSync` cases separately,
  documenting kvdb's structural exclusion.
- Retain these public errors: absent rename targets match `ErrAccountNotFound`;
  duplicate names, duplicate rename targets, and self-renames match
  `ErrAccountAlreadyExists`; already-locked creation matches
  `ErrStateForbidden`. Invalid request/field combinations match
  `ErrInvalidParam`; malformed, private, wrong-network, or wrong-depth account
  keys match `ErrInvalidAccountKey`; otherwise-valid unsupported backend or
  custody shapes match `ErrAccountOperationUnsupported`.
- Prove invalid request/key validation precedes unsupported-shape rejection.
  After validation, occupied names precede unsupported backend/custody errors,
  including exact-on-kvdb and supplied-XPub-on-unsupported-custody requests.
  Cover occupied names on all three backends and exact-number occupancy on
  SQLite and PostgreSQL. Duplicate names and occupied supported exact numbers
  precede overlapping XPub identities; otherwise overlapping XPub identities
  match `ErrAccountIdentityCollision`.
- Task 392 owns occupied-name precedence, Task 343 owns supported exact-number
  occupancy, Task 388 owns XPub collision admission, and Task 391 preserves
  their rules for supplied XPub custody. Retain 391's dependency for this final
  shape coverage; those production tasks keep their focused regressions.
- Match only documented Wallet error identities and compare the public account
  namespace before rejection, afterward, and after fresh Manager startup.
  Do not compare Store, legacy-manager, Vault, or SQL sentinels.
- Leave injected ambiguous-commit, concurrent lock/preparation, derivation-work,
  and terminal-exhaustion cases with their focused owners; Task 378 already
  proves locked creation rejects before Vault/Store work. Add no production
  fault seam, Store assertion, or new API. Fix only defects revealed by the
  required public cases and retain their regressions in this task.
- Task 429 retains general mutation success/reload and Task 430 the query
  contract. This runtime-shape evidence is not a legacy-conversion gate;
  Task 373 owns migration identity validation.

## Suggested fix

Upgrade existing rename assertions and extend the AccountManager integration
table only where the final rejection contract lacks evidence. Build collision
fixtures through public account creation, snapshot the public account view,
run the rejected mutation, and compare the same view before and after a fresh
reopen.

## Verification

- Run shared `RenameAccount` absence, duplicate-target, and self-rename cases,
  plus already-locked `NewAccount`, invalid-input, invalid-key, duplicate-name,
  and XPub-collision precedence cases on kvdb, SQLite, and PostgreSQL.
- Run exact-number, custom-scope, path-bound, and `NoChainSync` precedence and
  unsupported cases on each backend that can represent the prerequisite
  account shape. Include exact-on-kvdb and supplied-XPub-on-unsupported-custody
  requests whose names are already occupied.
- Prove every error matches only its documented wallet-owned identity and
  leaves the public account view unchanged before and after reopen.

## Definition of done

- The final real-backend-representable account mutation errors and precedence,
  including self-`RenameAccount` and locked `NewAccount`, have focused
  imported-`wallet` coverage across every applicable maintained backend.
- Modern kvdb's SQL-only exclusions are explicit, while shared cases contain
  no backend skip branch or backend-specific assertion.
- Existing integration cases own assertion upgrades, with no new API, Store
  assertion, or duplicated Task 429 coverage.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t391]: 391-path-bound-watch-only-xpub-accounts.md
[source]: https://github.com/btcsuite/btcwallet/blob/48e7f4b7e0f3808b5022fd817d12c6a4361bb9c1/itest/account_manager_test.go
