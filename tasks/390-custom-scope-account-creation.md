# Task 390 — create root-derived accounts in custom scopes

Component: **AccountManager custom scope and SQL key-scope/account mutation**.

Type: **Gap fix**.

Dependencies: **[Task 343][t343] and [Task 387][t387]**.

## Rationale

The Store can insert a new scope when given a schema, but public
root-derived-account creation never passes one and does not reject a
conflicting schema for an existing scope. Lnd needs exact purpose-1017
accounts in its custom scope.

## Details

- Add an optional scope schema to Task 392's `NewAccountParams` and require it
  when the scope is unknown; do not add another request or method.
- Require an exact account number for custom-scope creation in this task.
  Sequential invalid-child progress remains in Task 460.
- Accept only existing account-key-derivable branch schemas. Reject script,
  anchor, unknown, and unsupported forms before Vault or Store access.
- Reject purpose or coin components at or above the hardened boundary before
  adding `HardenedKeyStart`.
- If the scope exists, require the supplied schema to match exactly; never
  silently ignore a conflicting override.
- Return Task 378's canonical `ErrAccountOperationUnsupported` on modern kvdb
  after Task 392's read-only occupied-name lookup and before Vault preparation
  or Store mutation. Map invalid or conflicting scope/schema requests to Task
  378's existing `ErrInvalidParam` identity.
- Reuse Task 343 for exact numbering, `NoChainSync`, Vault preparation, and
  cursor CAS. Do not add a custom-scope cursor, result reader, or allocation
  component.
- On an ambiguous mutation, return Task 467's shared
  `ErrIndeterminateCommit` without retrying or reconciling the exact result.
  [Task 389][t389] later extends its exact result reader with this task's
  custom-scope presence and schema-equality facts.
- Keep supplied XPub/path-bound accounts in Task 391. Do not add a
  purpose-1017 special case, key-family API, or another scope registry.
- Keep implementation local to existing AccountManager, key-scope Store,
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

Pass one validated explicit schema through the existing key-scope mutation and
delegate account creation to the retained AccountManager and account CAS.

## Verification

Purpose-1017 exact accounts with both `NoChainSync` values survive reopen on
SQLite and PostgreSQL. Unknown scope without schema,
conflicting schema, unsupported branch types, and hardened-overflow components
fail before secret or Store mutation and match `ErrInvalidParam`. Modern kvdb
matches `ErrAccountOperationUnsupported` after the permitted occupied-name
lookup and before Vault preparation or Store mutation.
Nil-number requests fail before secret or Store mutation. Ambiguous-commit
injection matches `ErrIndeterminateCommit`, never retries, and publishes no
runtime account state.

## Definition of done

Exact root-derived accounts can use a truthful custom scope and explicit
schema through the generic API, without sequential invalid-child repair,
exact-result reconciliation, supplied XPub support, schema cleanup,
lnd-specific methods, or a new component.

[t343]: 343-exact-account-creation.md
[t378]: 378-account-manager-error-boundary.md
[t387]: 387-account-creation-identity-adr.md
[t389]: 389-runtime-account-commit-reconciliation.md
[t392]: 392-discovery-mode-account-creation.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
