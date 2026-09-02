# Task 378 — translate AccountManager backend errors

Component: **AccountManager public error boundary**.

Type: **Bug fix**.

Dependencies: **None**.

## Rationale

AccountManager currently returns `db.ErrAccountNotFound` and unwraps
`waddrmgr.ManagerError` directly. External callers therefore depend on a Store
or legacy-manager error contract that is not owned by wallet.

## Details

- Reuse the existing wallet-owned `ErrAccountNotFound`,
  `ErrInvalidAccountKey`, `ErrInvalidParam`, and `ErrStateForbidden`
  identities. Define only the AccountManager-specific
  `ErrAccountAlreadyExists`, `ErrAccountOperationUnsupported`, and terminal
  `ErrAccountDerivationExhausted` identities needed by currently callable
  operations; do not export an internal error table wholesale.
- Map the complete currently supported AccountManager outcomes exactly:
  successful list queries with no matches return an empty result and nil
  error; absent `GetAccount` or `RenameAccount` targets return
  `ErrAccountNotFound`; duplicate names or rename targets return
  `ErrAccountAlreadyExists`; renaming an existing account to its current name
  also returns `ErrAccountAlreadyExists` before mutation; invalid request
  combinations return `ErrInvalidParam`; invalid account keys retain
  `ErrInvalidAccountKey`;
  watch-only `NewAccount` and spendable-SQL `ImportAccount` return
  `ErrAccountOperationUnsupported` instead of leaking
  `errWatchOnlyAccountDerivation` or
  `db.ErrSpendableWalletNeedsAccountPrivKey`; and terminal sequential
  `NewAccount` exhaustion returns `ErrAccountDerivationExhausted` instead of
  leaking `db.ErrMaxAccountNumberReached` or
  `waddrmgr.ErrAccountNumTooHigh`.
- After lifecycle, request-shape, and key-material validation, resolve an
  occupied name before mutation. Database constraint or lookup order must not
  change that public identity.
- Preserve `context.Canceled`, `context.DeadlineExceeded`, and existing
  lifecycle `ErrStateForbidden` results without translating them. In
  particular, `NewAccount` rejects an already-locked wallet before Vault
  preparation and translates a concurrent `keyvault.ErrVaultLocked` result to
  `ErrStateForbidden`; the internal Vault sentinel never escapes.
- Preserve useful operation and selector context while wrapping only the
  wallet-owned sentinel with `%w`. Do not return `db` sentinels or
  `waddrmgr.ManagerError` values from public methods. Unexpected internal
  failures retain diagnostic context but match no additional stable sentinel.
- Keep future exact-number occupancy and precedence in [Task 343][t343],
  normalized-XPub collision identity and admission in [Task 388][t388], and
  future valid unsupported request shapes in [Task 392][t392]. Those later
  paths reuse this task's canonical unsupported identity; this dependency-free
  task does not test request paths that they have not activated.
- Keep the shared durable-mutation identity in [Task 467][t467], address-child
  exhaustion in [Task 344][t344] for reuse by [Task 395][t395], and durable
  invalid-child progress plus the bounded per-call limit in [Task 460][t460].
  Task 460 reuses this task's terminal account-exhaustion identity when it
  persists that existing outcome; it does not define another sentinel.
- Cross-link [Task 430][t430] as the existing imported-`wallet` query owner and
  [Task 462][t462] as the later public mutation-error and precedence owner.
  This task keeps focused translation and lock coverage rather than
  duplicating those real-backend suites.
- Keep duplicate wallet creation in Task 330, DTO conversion in Task 377,
  exact root-derived behavior in [Task 343][t343], and custom-scope behavior in
  [Task 390][t390].
- Keep implementation local to existing AccountManager error translation,
  source files, and colocated tests. Add no parallel interface, arbitrary
  helper, production file, or standalone test file; justify any unavoidable
  new file against the repository's current layout in the implementation PR.
- Document every added code block inline with why it exists and how it works;
  comments that merely restate the code do not satisfy this requirement.
- Give every unit test detailed Arrange, Act, and Assert comments explaining
  its setup, operation, and evidence. Interface mocks embed `mock.Mock`, set
  `On(...).Return(...)` expectations in Arrange with the narrowest cardinality
  (`Once()` for one required call), and call `AssertExpectations(t)` directly
  in Assert without duplicate call assertions or a `require.True` wrapper.

## Suggested fix

Add a small AccountManager-specific translation function keyed only by the
exhaustive stable account outcomes above. Call it once at each Store,
legacy-manager, or Vault exit and remove the current
`errors.As(..., waddrmgr.ManagerError)` passthrough.

## Verification

- Focused colocated tests inject Store, legacy-manager, Vault, and lifecycle
  outcomes for every currently callable mapping. An already-locked
  `NewAccount` stops before Vault or Store work, while a
  lock-during-preparation result matches `ErrStateForbidden` without matching
  `keyvault.ErrVaultLocked`. Watch-only derivation and spendable-SQL import
  refusal both match `ErrAccountOperationUnsupported` without matching their
  internal errors. Current SQL and kvdb terminal account-range failures both
  match `ErrAccountDerivationExhausted` without matching their backend errors.
- Colocated translation tests require self-rename to match
  `ErrAccountAlreadyExists` before mutation. Task 462 proves the same public
  identity and durable no-change result on all three maintained backends.
- Validation precedes occupied-name lookup, and unexpected injected failures
  match no listed wallet identity.
- Task 430 proves public query absence, and Task 462 proves the final public
  mutation outcomes after their activating tasks land.

## Definition of done

- AccountManager exposes only wallet-owned stable errors for its supported
  outcomes, with equivalent SQL and kvdb conditions sharing one `errors.Is`
  result.
- Currently callable validation, occupied-name, self-rename, absence,
  unsupported, exhaustion, and lifecycle precedence is deterministic.
- No Manager, AddressManager, genesis, recovery, or future bounded-work
  derivation-limit contract is introduced into this task.

[t343]: 343-exact-account-creation.md
[t344]: 344-bulk-address-allocation.md
[t388]: 388-runtime-account-xpub-collisions.md
[t390]: 390-custom-scope-account-creation.md
[t392]: 392-discovery-mode-account-creation.md
[t395]: 395-key-only-locator-allocation.md
[t460]: 460-sequential-account-invalid-children.md
[t462]: 462-account-mutation-error-itests.md
[t430]: 430-account-query-itests.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
