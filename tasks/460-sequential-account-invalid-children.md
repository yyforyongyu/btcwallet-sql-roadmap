# Task 460 — make sequential account allocation invalid-child safe

Component: **AccountManager sequential creation and SQL AccountStore cursor**.

Type: **Bug fix**.

Dependencies: **[Task 378][t378] and [Task 467][t467]**.

## Rationale

A genuinely invalid account child can make sequential SQL account creation
retry the same unusable number. This is a real but extraordinarily rare
derivation edge case, not an observed normal-operation blocker; its protection
belongs in Hardening.

## Details

- Skip only genuinely invalid children at the sequential account-derivation
  step, using checked account-range bounds and the existing cursor.
- Do not treat arbitrary derivation failures or later branch failures as proof
  that the account child is invalid. Do not substitute another number for an
  exact request.
- Preserve account/cursor atomicity, concurrent allocation, and the existing
  `ErrAccountDerivationExhausted` identity without wrapping the account range.
- Preserve conservative `ErrIndeterminateCommit` reporting: no automatic
  retry, readback reconciliation, or uncertain success.
- Add no per-call work-limit policy, `ErrAccountDerivationLimit`,
  cursor-only progress protocol, or extra branch-validation requirement.
  Keep modern kvdb behavior unchanged.

## Suggested fix

Handle invalid sequential account children at the existing allocation
boundary, preserving checked cursor progression and existing exhaustion
handling. Leave private derivation and transaction mechanics open.

## Verification

- Inject an invalid account child on SQLite and PostgreSQL, then a valid one;
  the returned account uses the valid number without reusing the skipped one.
- Arbitrary errors and exact-number failures do not silently select another
  account.
- Boundary and concurrent cases preserve unique progression and the existing
  exhaustion identity; ambiguity returns no account and is never retried.

## Definition of done

Sequential SQL allocation can skip a genuinely invalid account child without
broadening error policy, adding a work-limit API, or redesigning derivation.

[t378]: ../archive/tasks/378-account-manager-error-boundary.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
