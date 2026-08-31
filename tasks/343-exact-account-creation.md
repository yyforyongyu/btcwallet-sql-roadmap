# Task 343 — create next-or-exact root-derived accounts

Component: **AccountManager root-derived creation and SQL AccountStore CAS**.

Type: **Gap fix**.

Dependencies: **[Task 332][t332], [Task 377][t377], [Task 378][t378],
[Task 392][t392], [Task 393][t393], [Task 452][t452], and [Task 467][t467]**.

## Rationale

The maintained account API cannot create a sparse exact account. Lnd needs
exact purpose-1017 account numbers without filler accounts, while custom-scope
and supplied-XPub identity remain independent extensions.

## Details

- Add an optional exact account number to Task 392's `NewAccountParams`. Nil
  preserves sequential creation. Activate Task 392's existing `NoChainSync`
  true value for SQL root-derived next-or-exact creation only after this task's
  receiving and recovery guards are complete; do not add an lnd-specific entry
  point, another request, or a positional compatibility signature.
- Limit this task to existing canonical scopes and wallet-root-derived account
  material. Reject schema overrides and supplied account keys until their
  follow-ups land.
- Validate each account component at or below the public maximum before adding
  the hardened bit or touching Vault or Store.
- For exact creation, require a vacant number and atomically update the cursor
  to checked `max(current, exact+1)`. Lower sparse holes remain available for
  later exact creation. An invalid exact child leaves account and cursor
  unchanged.
- After lifecycle and request validation, resolve an occupied name before an
  occupied exact number and before mutation. Both conditions wrap Task 378's
  `ErrAccountAlreadyExists`; runtime XPub-collision admission remains in Task
  388 and follows those checks when it lands.
- Prepare the account key, both branches, ciphertext, and public facts through
  the maintained Vault and derivation helpers before opening the Store write.
- Persist the account, encrypted account secret, fingerprint, `NoChainSync`,
  and cursor in one method-specific SQL transaction.
- On `runtime.ErrAmbiguousTxCommit`, return Task 467's wallet-owned
  `ErrIndeterminateCommit` and never retry from the transport error. Task 389
  owns exact result reconciliation.
- Do not create an initial live-watch obligation: an account row exposes no
  receiving script. Address allocation owns the first script.
- Preserve modern kvdb's existing sequential root-derived subset only when
  `NoChainSync` is false. Preserve Task 392's validation and occupied-name
  precedence, then return Task 378's canonical
  `ErrAccountOperationUnsupported` before mutation for an exact number or
  another SQL-only extension; do not change its format.
- Keep sequential invalid-child cursor repair in [Task 460][t460]. Exclude
  custom scopes, supplied XPub accounts, Manager genesis, watch delivery, and
  generic reconciliation.
- Keep implementation local to existing components, source files, and
  colocated test files. Add no parallel interface, arbitrary helper,
  production file, or standalone test file; justify any unavoidable new file
  against the repository's existing layout in the implementation PR.
- Document every added code block inline with why it exists and how it works;
  comments that merely restate the code do not satisfy this requirement.
- Give every unit test detailed Arrange, Act, and Assert comments explaining
  its setup, operation, and evidence. Interface mocks embed `mock.Mock`, set
  `On(...).Return(...)` expectations in Arrange with the narrowest cardinality
  (`Once()` for one required call), and call `AssertExpectations(t)` directly
  in Assert without duplicate call assertions or a `require.True` wrapper.

## Suggested fix

Extend the existing account request and method-specific SQL CAS with an
optional exact number while reusing Task 392's `NoChainSync`, the maintained
Vault, and the account cursor.

## Verification

SQLite and PostgreSQL cover sequential and sparse exact creation, lower-hole
fill, bounds, concurrent exact/sequential requests, invalid exact children,
both `NoChainSync` values, restart, and deterministic validation, occupied-name,
occupied-number, then mutation precedence. Occupied names and numbers both
match `ErrAccountAlreadyExists`; ambiguous returns never retry.
Modern-kvdb fixtures preserve the existing sequential subset and prove exact
requests fail before Vault or Store mutation and satisfy
`errors.Is(err, wallet.ErrAccountOperationUnsupported)`; Task 392 owns the
corresponding `NoChainSync` rejection. Spies prove derivation and Vault work
finish before the Store write opens.

## Definition of done

Canonical SQL scopes support one final next-or-exact root-derived operation
with atomic account, secret, chain-sync, and cursor facts. Modern kvdb retains
its existing sequential subset, and no custom-scope, supplied-XPub, genesis,
watch-delivery, or sequential invalid-child repair is included.

[t332]: ../archive/tasks/332-watch-only-script-policy.md
[t377]: ../archive/tasks/377-account-manager-wallet-dto.md
[t378]: 378-account-manager-error-boundary.md
[t392]: 392-discovery-mode-account-creation.md
[t393]: 393-key-only-receiving-rejection.md
[t452]: 452-key-only-recovery-derivation.md
[t460]: 460-sequential-account-invalid-children.md
[t467]: 467-indeterminate-commit-identity.md
