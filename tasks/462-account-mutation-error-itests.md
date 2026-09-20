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

- Extend existing rejection cases for missing identities and precedence; do
  not duplicate ordinary creation cases already using `errors.Is`. Add only
  missing rename, supplied-XPub, and collision evidence. Retain Task 391 as
  the prerequisite for the complete supported-shape contract.
- Upgrade `testAccountManagerRejectAccountRename`, the locked-create case in
  `testAccountManagerEnforceAccountCreationLifecycle`, and generic assertions
  in `testAccountManagerRejectAccountImport` in place. Preserve existing
  nil/private-key, stopped, and basic-create assertions without another case
  registration. Add a case only for a shape or precedence not yet exercised.
- This later-feature verification remains in Phase 2, but is not a prerequisite
  for converting legacy account shapes. Task 373 owns migration identity
  validation. An enabled runtime path must still satisfy its required errors.
- Exercise only public AccountManager mutation APIs through the imported
  `wallet` package on real kvdb, SQLite, and PostgreSQL Wallets.
- Register shared cases on all three backends whenever they can represent the
  same request. Keep exact, custom-scope, path-bound, and `NoChainSync`
  SQL-only cases separate, and document modern kvdb's structural exclusion
  instead of skipping it inside a shared contract.
- Cover absent `RenameAccount` targets matching `ErrAccountNotFound`, rename
  targets colliding with an existing name matching `ErrAccountAlreadyExists`,
  self-renames matching `ErrAccountAlreadyExists`, and an already-locked
  `NewAccount` matching `ErrStateForbidden` on kvdb, SQLite, and PostgreSQL.
  Use public queries to prove each rejected call leaves the account namespace
  unchanged; Task 378's injected tests alone prove that locked creation stops
  before Vault or Store work.
- Cover the final representable mutation identities and precedence:
  invalid request or field combinations match `ErrInvalidParam`; malformed,
  private, wrong-network, or wrong-depth account keys match
  `ErrInvalidAccountKey` (reuse existing nil/private-key cases);
  duplicate names and occupied exact numbers beat overlapping XPub
  identities; otherwise-equal overlapping XPub identities match
  `ErrAccountIdentityCollision`; and valid unsupported backend or custody
  shapes match `ErrAccountOperationUnsupported`.
- Combine invalid request or key material with an otherwise unsupported shape
  to prove validation runs first. After valid shape and key validation, require
  an occupied name to match `ErrAccountAlreadyExists` before unsupported
  backend or custody classification. Require occupied exact-number precedence
  only on SQLite and PostgreSQL paths where Task 343 supports exact creation.
  Cover an occupied-name collision on all three backends, Task 343's occupied
  exact-number collision on SQLite and PostgreSQL, an occupied-name exact
  request on modern kvdb, and an occupied-name supplied-XPub request on
  unsupported wallet custody. Task 392 owns occupied-name-before-unsupported
  ordering, Task 343 owns supported exact-number occupancy, Task 388 owns XPub
  collision ordering, and Task 391 preserves the shared rule for its custody
  checks.
- Use public account queries before rejection and after durable reopen to prove
  each error leaves the account namespace unchanged. Assert identities only
  with `errors.Is`; never expose or compare Store, legacy-manager, Vault, or
  SQL sentinels.
- Leave ambiguous-commit, concurrent lock-during-preparation races, derivation
  work limits, and terminal range exhaustion in their owning colocated fault
  tests when a real public fixture cannot represent them deterministically. Do
  not add a production fault seam merely to force those cases into an
  integration test.
- Do not repeat Task 429's general success, result-field, or reload matrix or
  Task 430's query contract. Fix only a production defect required by the
  listed public cases, and include that regression in this task's
  implementation PR.
- Extend the existing `itest/account_manager_test.go` suite. Add no arbitrary
  helper, production file, or standalone test file; justify any unavoidable
  new file against the repository's current layout in the implementation PR.

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
- The existing integration-test suite owns the cases without a new API,
  direct Store assertion, arbitrary file, or duplicated Task 429 coverage.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t391]: 391-path-bound-watch-only-xpub-accounts.md
[source]: https://github.com/btcsuite/btcwallet/blob/48e7f4b7e0f3808b5022fd817d12c6a4361bb9c1/itest/account_manager_test.go
