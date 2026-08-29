# Task 390 — create root-derived accounts in custom scopes

Component: **AccountManager custom scope and SQL key-scope/account mutation**.

Type: **Gap fix**.

Dependencies: **[Task 343][t343], [Task 387][t387], [Task 389][t389], and
[Task 460][t460]**.

## Rationale

The Store can insert a new scope when given a schema, but public
root-derived-account creation never passes one and does not reject a
conflicting schema for an existing scope. Lnd needs exact purpose-1017
accounts in its custom scope.

## Details

- Activate Task 392's dormant optional scope schema in the final
  `NewAccountParams` and require it when the scope is unknown; do not redefine
  the request.
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
- Reuse Task 343 for nil/exact numbering, `NoChainSync`, Vault preparation,
  collision admission, cursor CAS, and account-specific reconciliation.
- For nil account numbers in a custom scope, apply Task 460's bounded
  invalid-child skip, durable cursor-progress, per-call limit, terminal
  exhaustion, and ambiguous-result reconciliation contracts to that scope's
  existing account cursor. Do not add a custom-scope cursor, result reader, or
  allocation component.
- Extend Task 389's completed result reader only with the custom scope's exact
  schema presence and equality facts. Never retry an ambiguous mutation.
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

Purpose-1017 exact and sequential accounts with both `NoChainSync` values
survive reopen on SQLite and PostgreSQL. Unknown scope without schema,
conflicting schema, unsupported branch types, and hardened-overflow components
fail before secret or Store mutation and match `ErrInvalidParam`. Modern kvdb
matches `ErrAccountOperationUnsupported` after the permitted occupied-name
lookup and before Vault preparation or Store mutation.
Custom-scope sequential cases inject invalid account and required branch
children, cross the private work bound, resume from the durable cursor, and
reach stable terminal exhaustion without repeating a child. Concurrent and
restart cases preserve unique progress. Ambiguous-commit injection covers
inserted account, cursor-only progress, terminal exhaustion, absent state, and
conflicting scope/account state without retry.

## Definition of done

Root-derived accounts can use a truthful custom scope and explicit schema
through the final generic API. Sequential custom-scope allocation inherits the
accepted bounded-progress and exhaustion policy without supplied XPub support,
schema cleanup, lnd-specific methods, or a new component.

[t343]: 343-exact-account-creation.md
[t378]: 378-account-manager-error-boundary.md
[t387]: 387-account-creation-identity-adr.md
[t389]: 389-runtime-account-commit-reconciliation.md
[t392]: 392-discovery-mode-account-creation.md
[t460]: 460-sequential-account-invalid-children.md
