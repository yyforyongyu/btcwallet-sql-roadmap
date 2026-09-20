# Task 460 — make sequential account allocation invalid-child safe

Component: **AccountManager sequential creation and SQL AccountStore cursor
CAS**.

Type: **Bug fix**.

Dependencies: **[Task 378][t378], [Task 389][t389], and [Task 467][t467]**.

## Rationale

Sequential root-derived account creation rolls back to the same cursor when an
account or required branch child is invalid. Every retry can select the same
unusable child and permanently wedge account allocation.

## Details

- Limit the fix to the existing sequential root-derived account operation.
  Exact-number policy, custom scopes, supplied XPub accounts, and API
  reshaping remain in their owning tasks.
- Skip and durably consume invalid account or required branch children through
  the existing checked account cursor and method-specific SQL CAS.
- Bound each call's derivation work and persist terminal exhaustion so retry
  cannot loop forever or wrap the public account range.
- If the private work bound is reached after consuming invalid children but
  before finding a valid account or terminal exhaustion, commit the cursor
  immediately after the last proven invalid child, insert no account, and
  define and return the operation-specific wallet-owned
  `ErrAccountDerivationLimit`. A retry is safe and resumes from that durable
  cursor; this is not terminal exhaustion.
- Persist terminal account-range exhaustion with no account insertion and
  return Task 378's `ErrAccountDerivationExhausted`. This is a stable terminal
  result; retries return the same identity without derivation or cursor work.
- Preserve account, branch, and cursor atomicity. A definite failure before
  commit leaves no partial account but retains only cursor progress that the
  accepted invalid-child policy explicitly requires.
- Reuse Task 389's same-AccountManager serialization through classification
  and extend its operation-specific result reader for the new facts. An exact
  inserted account returns success; exact cursor-only bounded progress returns
  `ErrAccountDerivationLimit`; exact persisted terminal exhaustion returns
  `ErrAccountDerivationExhausted`; absence returns ordinary failure; and
  partial, conflicting, or otherwise unprovable state returns Task 467's
  `ErrIndeterminateCommit`. Never retry from `runtime.ErrAmbiguousTxCommit` or
  add replay machinery.
- Preserve modern kvdb behavior and storage. Task 382 may later move
  derivation behind the Vault without changing this allocation policy.
- Keep implementation local to existing AccountManager, account cursor Store,
  source files, and colocated tests. Add no parallel interface, arbitrary
  helper, production file, or standalone test file; justify any unavoidable
  new file against the repository's current layout in the implementation PR.

## Suggested fix

Extend the existing sequential account cursor transaction to record bounded
invalid-child skips, work-limit progress, and exhaustion while keeping account
derivation outside the write. Extend Task 389's private boundary and exact
result reader only for these new sequential outcomes.

## Verification

SQLite and PostgreSQL tests inject invalid account, external-branch, and
internal-branch children and prove later valid accounts remain allocatable
without reusing a skipped child. Boundary and restart cases prove exhaustion
is durable, concurrent calls preserve unique progression, and an ambiguous
result never triggers a retry. Multi-call and restart cases exhaust the
private work limit, match `ErrAccountDerivationLimit`, prove no account was
inserted, and resume from the committed cursor until a valid account or stable
terminal `ErrAccountDerivationExhausted`. Ambiguous inserted-account,
cursor-only progress, and terminal-exhaustion cases cover exact, absent, and
conflicting state without retry or same-AccountManager misattribution. Modern
kvdb behavior remains unchanged.

## Definition of done

Sequential SQL account allocation cannot wedge on an invalid child, repeat
work after its per-call bound, or retry forever at exhaustion. Limit errors
preserve durable forward progress, and the focused fix adds no exact-account
feature, reconciliation system, API redesign, new component, or arbitrary
file.

[t378]: ../archive/tasks/378-account-manager-error-boundary.md
[t389]: 389-runtime-account-commit-reconciliation.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
